# capybot
name: Build APK
on:
  workflow_dispatch:
  push:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Write project files
        run: |
          mkdir -p app app/src/main app/src/main/java/com/example/capybot app/src/main/res/values app/src/main/res/xml
          cat > app/build.gradle.kts <<'EOF_CAPY'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }

          android {
              namespace = "com.example.capybot"
              compileSdk = 34

              defaultConfig {
                  applicationId = "com.example.capybot"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }

              buildTypes {
                  release {
                      isMinifyEnabled = false
                  }
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions {
                  jvmTarget = "17"
              }
          }

          dependencies {
              implementation("androidx.core:core-ktx:1.13.1")
              implementation("org.opencv:opencv:4.9.0")
          }
          EOF_CAPY
          cat > app/src/main/AndroidManifest.xml <<'EOF_CAPY'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

              <application
                  android:allowBackup="false"
                  android:label="Capy Bot"
                  android:theme="@android:style/Theme.Material.Light.NoActionBar">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <!-- Non-root mode: screen capture (MediaProjection) + taps via Accessibility -->
                  <service
                      android:name=".BotService"
                      android:exported="false"
                      android:foregroundServiceType="mediaProjection" />

                  <!-- Root mode: screencap/input via su -->
                  <service
                      android:name=".RootBotService"
                      android:exported="false"
                      android:foregroundServiceType="specialUse">
                      <property
                          android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                          android:value="Personal automation of on-screen taps" />
                  </service>

                  <service
                      android:name=".BotAccessibilityService"
                      android:exported="true"
                      android:label="Capy Bot"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifest>
          EOF_CAPY
          cat > app/src/main/java/com/example/capybot/BotAccessibilityService.kt <<'EOF_CAPY'
          package com.example.capybot

          import android.accessibilityservice.AccessibilityService
          import android.accessibilityservice.GestureDescription
          import android.content.Intent
          import android.graphics.Path
          import android.view.accessibility.AccessibilityEvent

          /** Performs taps for the non-root mode. Must be enabled by the user in Accessibility settings. */
          class BotAccessibilityService : AccessibilityService() {

              companion object {
                  @Volatile
                  var instance: BotAccessibilityService? = null
              }

              override fun onServiceConnected() {
                  instance = this
              }

              override fun onUnbind(intent: Intent?): Boolean {
                  instance = null
                  return super.onUnbind(intent)
              }

              override fun onAccessibilityEvent(event: AccessibilityEvent?) {}

              override fun onInterrupt() {}

              fun tap(x: Int, y: Int): Boolean {
                  val path = Path().apply { moveTo(x.toFloat(), y.toFloat()) }
                  val gesture = GestureDescription.Builder()
                      .addStroke(GestureDescription.StrokeDescription(path, 0, 60))
                      .build()
                  return dispatchGesture(gesture, null, null)
              }
          }
          EOF_CAPY
          cat > app/src/main/java/com/example/capybot/BotService.kt <<'EOF_CAPY'
          package com.example.capybot

          import android.app.Notification
          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.Service
          import android.content.ContentValues
          import android.content.Intent
          import android.content.pm.ServiceInfo
          import android.graphics.Bitmap
          import android.graphics.BitmapFactory
          import android.graphics.PixelFormat
          import android.hardware.display.DisplayManager
          import android.hardware.display.VirtualDisplay
          import android.media.ImageReader
          import android.media.projection.MediaProjection
          import android.media.projection.MediaProjectionManager
          import android.os.Build
          import android.os.Handler
          import android.os.IBinder
          import android.os.Looper
          import android.provider.MediaStore
          import android.util.DisplayMetrics
          import android.util.Log
          import android.view.WindowManager
          import org.opencv.android.OpenCVLoader
          import java.io.File

          /**
           * Foreground service that runs the bot loop:
           * screenshot -> find known images -> Policy picks one -> tap -> repeat.
           *
           * BotService      = non-root mode (MediaProjection screenshots + Accessibility taps)
           * RootBotService  = root mode (su screencap + su input tap)
           */
          open class BotService : Service() {

              companion object {
                  const val EXTRA_RESULT_CODE = "result_code"
                  const val EXTRA_DATA = "data"
                  const val LOOP_DELAY_MS = 1500L
                  const val AFTER_TAP_DELAY_MS = 1000L
                  const val MAX_REFRESH_STREAK = 3
                  const val STUCK_REPEATS = 8
                  const val BLOCK_MS = 20_000L
                  const val IDLE_LOOPS_BEFORE_SAVE = 25
                  const val MAX_UNKNOWN_SHOTS = 10
                  private const val CHANNEL_ID = "capybot"
                  private const val TAG = "CapyBot"

                  @Volatile
                  var running = false
              }

              protected open val rootMode = false

              @Volatile
              private var stopRequested = false
              private var worker: Thread? = null

              private var projection: MediaProjection? = null
              private var reader: ImageReader? = null
              private var virtualDisplay: VirtualDisplay? = null
              private var width = 0
              private var height = 0
              private var lastShot: Bitmap? = null

              // loop state
              private val blocked = HashMap<String, Long>()
              private var lastKey: String? = null
              private var repeatCount = 0
              private var refreshStreak = 0
              private var idleLoops = 0
              private var savedUnknown = 0

              override fun onBind(intent: Intent?): IBinder? = null

              override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
                  if (running) return START_NOT_STICKY
                  startInForeground()

                  if (!OpenCVLoader.initLocal()) {
                      Log.e(TAG, "OpenCV failed to initialize")
                      stopSelf()
                      return START_NOT_STICKY
                  }

                  if (!rootMode) {
                      val resultCode = intent?.getIntExtra(EXTRA_RESULT_CODE, 0) ?: 0
                      @Suppress("DEPRECATION")
                      val data: Intent? = intent?.getParcelableExtra(EXTRA_DATA)
                      if (data == null || !setupProjection(resultCode, data)) {
                          stopSelf()
                          return START_NOT_STICKY
                      }
                  }

                  running = true
                  stopRequested = false
                  worker = Thread { loop() }.also { it.start() }
                  return START_NOT_STICKY
              }

              override fun onDestroy() {
                  stopRequested = true
                  running = false
                  worker?.interrupt()
                  virtualDisplay?.release()
                  reader?.close()
                  projection?.stop()
                  super.onDestroy()
              }

              // ---------- foreground notification ----------

              private fun startInForeground() {
                  val nm = getSystemService(NOTIFICATION_SERVICE) as NotificationManager
                  nm.createNotificationChannel(
                      NotificationChannel(CHANNEL_ID, "Capy Bot", NotificationManager.IMPORTANCE_LOW)
                  )
                  val notification = Notification.Builder(this, CHANNEL_ID)
                      .setContentTitle("Capy Bot is running")
                      .setContentText("Open the app to stop it")
                      .setSmallIcon(android.R.drawable.ic_media_play)
                      .setOngoing(true)
                      .build()

                  if (rootMode) {
                      if (Build.VERSION.SDK_INT >= 34) {
                          startForeground(1, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)
                      } else {
                          startForeground(1, notification)
                      }
                  } else {
                      if (Build.VERSION.SDK_INT >= 29) {
                          startForeground(1, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PROJECTION)
                      } else {
                          startForeground(1, notification)
                      }
                  }
              }

              // ---------- non-root: screen capture ----------

              private fun setupProjection(resultCode: Int, data: Intent): Boolean {
                  return try {
                      val dm = DisplayMetrics()
                      val wm = getSystemService(WINDOW_SERVICE) as WindowManager
                      @Suppress("DEPRECATION")
                      wm.defaultDisplay.getRealMetrics(dm)
                      width = dm.widthPixels
                      height = dm.heightPixels

                      val mpm = getSystemService(MEDIA_PROJECTION_SERVICE) as MediaProjectionManager
                      val proj = mpm.getMediaProjection(resultCode, data)
                      // Android 14+ requires a callback registered before creating the virtual display.
                      proj.registerCallback(object : MediaProjection.Callback() {
                          override fun onStop() {
                              stopRequested = true
                              stopSelf()
                          }
                      }, Handler(Looper.getMainLooper()))
                      projection = proj

                      reader = ImageReader.newInstance(width, height, PixelFormat.RGBA_8888, 2)
                      virtualDisplay = proj.createVirtualDisplay(
                          "capybot", width, height, dm.densityDpi,
                          DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
                          reader!!.surface, null, null
                      )
                      true
                  } catch (e: Exception) {
                      Log.e(TAG, "Projection setup failed", e)
                      false
                  }
              }

              private fun projectionScreenshot(): Bitmap? {
                  val r = reader ?: return lastShot
                  val image = r.acquireLatestImage() ?: return lastShot // no new frame = screen unchanged
                  try {
                      val plane = image.planes[0]
                      val rowPadding = plane.rowStride - plane.pixelStride * width
                      val padded = Bitmap.createBitmap(
                          width + rowPadding / plane.pixelStride, height, Bitmap.Config.ARGB_8888
                      )
                      padded.copyPixelsFromBuffer(plane.buffer)
                      val shot = Bitmap.createBitmap(padded, 0, 0, width, height)
                      if (padded !== shot) padded.recycle()
                      lastShot = shot
                      return shot
                  } finally {
                      image.close()
                  }
              }

              // ---------- root mode ----------

              private fun rootScreenshot(): Bitmap? {
                  return try {
                      val p = Runtime.getRuntime().exec(arrayOf("su", "-c", "screencap -p"))
                      val bytes = p.inputStream.readBytes()
                      p.waitFor()
                      BitmapFactory.decodeByteArray(bytes, 0, bytes.size)
                  } catch (e: Exception) {
                      Log.e(TAG, "Root screenshot failed", e)
                      null
                  }
              }

              private fun rootTap(x: Int, y: Int): Boolean {
                  return try {
                      Runtime.getRuntime().exec(arrayOf("su", "-c", "input tap $x $y")).waitFor() == 0
                  } catch (e: Exception) {
                      false
                  }
              }

              // ---------- saving screens the bot did not understand ----------

              /** Saves to Pictures/CapyBot (Android 10+) so you can crop new button images from it. */
              private fun saveUnknown(bmp: Bitmap) {
                  if (Build.VERSION.SDK_INT < 29 || savedUnknown >= MAX_UNKNOWN_SHOTS) return
                  try {
                      val values = ContentValues().apply {
                          put(MediaStore.Images.Media.DISPLAY_NAME, "capybot_unknown_${System.currentTimeMillis()}.png")
                          put(MediaStore.Images.Media.MIME_TYPE, "image/png")
                          put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CapyBot")
                      }
                      val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values) ?: return
                      contentResolver.openOutputStream(uri)?.use { bmp.compress(Bitmap.CompressFormat.PNG, 100, it) }
                      savedUnknown++
                      Log.i(TAG, "Saved unknown screen #$savedUnknown to Pictures/CapyBot")
                  } catch (e: Exception) {
                      Log.e(TAG, "Could not save unknown screen", e)
                  }
              }

              // ---------- main loop ----------

              private fun tap(x: Int, y: Int): Boolean =
                  if (rootMode) rootTap(x, y) else (BotAccessibilityService.instance?.tap(x, y) ?: false)

              private fun loop() {
                  val settings = Settings.load(this)
                  val templates = Matcher.loadTemplates(File(filesDir, "templates"))
                  val hpRegion = HpReader.parseRegion(settings.hpRegion)
                  Log.i(TAG, "Loaded ${templates.size} templates (rootMode=$rootMode, hpRegion=${hpRegion != null})")
                  if (templates.isEmpty()) {
                      stopSelf()
                      return
                  }
                  try {
                      while (!stopRequested) {
                          val now = System.currentTimeMillis()
                          blocked.entries.removeAll { it.value < now }

                          val shot = if (rootMode) rootScreenshot() else projectionScreenshot()
                          var acted = false
                          if (shot != null) {
                              val hits = Matcher.findAll(shot, templates, HashSet(blocked.keys))
                              val hpRatio = hpRegion?.let { HpReader.fillRatio(shot, it) }
                              val hpLow = hits.any { Policy.isLowHpMarker(it) } ||
                                  (hpRatio != null && hpRatio < HpReader.LOW_RATIO)
                              val refreshAllowed = settings.useRefresh && refreshStreak < MAX_REFRESH_STREAK

                              val choice = Policy.decide(hits, hpLow, refreshAllowed)
                              if (choice != null) {
                                  Log.i(
                                      TAG,
                                      "${choice.name} (${"%.2f".format(choice.score)}) -> tap ${choice.x},${choice.y}" +
                                          " hp=${hpRatio?.let { "%.2f".format(it) } ?: "n/a"} low=$hpLow"
                                  )
                                  tap(choice.x, choice.y)
                                  acted = true
                                  refreshStreak = if (Policy.isRefresh(choice)) refreshStreak + 1 else 0

                                  // stuck detection: same thing tapped at the same spot many times in a row
                                  val k = "${choice.name}@${choice.x / 30},${choice.y / 30}"
                                  if (k == lastKey) repeatCount++ else {
                                      lastKey = k
                                      repeatCount = 1
                                  }
                                  if (repeatCount >= STUCK_REPEATS) {
                                      Log.w(TAG, "Stuck on ${choice.name}; ignoring it for ${BLOCK_MS / 1000}s")
                                      blocked[choice.name] = now + BLOCK_MS
                                      repeatCount = 0
                                  }
                                  Thread.sleep(AFTER_TAP_DELAY_MS)
                              }
                          }

                          if (acted) {
                              idleLoops = 0
                          } else {
                              idleLoops++
                              if (idleLoops >= IDLE_LOOPS_BEFORE_SAVE) {
                                  shot?.let { saveUnknown(it) }
                                  idleLoops = 0
                              }
                          }
                          Thread.sleep(LOOP_DELAY_MS)
                      }
                  } catch (_: InterruptedException) {
                  }
              }
      
