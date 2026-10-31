# Jarvis-<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>
    <uses-permission android:name="android.permission.CALL_PHONE"/>
    <uses-permission android:name="android.permission.CAMERA"/>
    <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"/>

    <application android:label="JARVIS" android:theme="@style/Theme.AppCompat.DayNight.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <service android:name=".JarvisService"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
            android:exported="true">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService"/>
            </intent-filter>
            <meta-data android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_config"/>
        </service>
    </application>
</manifest><accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeAllMask"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:canRetrieveWindowContent="true"
    android:canPerformGestures="true"
    android:accessibilityFlags="flagDefault"/>package com.jarvis.app

import android.accessibilityservice.AccessibilityService
import android.view.accessibility.AccessibilityEvent
import android.view.accessibility.AccessibilityNodeInfo

class JarvisService : AccessibilityService() {
    companion object { var instance: JarvisService? = null }

    override fun onServiceConnected() { instance = this }
    override fun onAccessibilityEvent(e: AccessibilityEvent?) {}
    override fun onInterrupt() {}

    fun clickText(text: String): Boolean {
        val node = rootInActiveWindow
            ?.findAccessibilityNodeInfosByText(text)?.firstOrNull() ?: return false
        var c: AccessibilityNodeInfo? = node
        while (c != null && !c.isClickable) c = c.parent
        return c?.performAction(AccessibilityNodeInfo.ACTION_CLICK) == true
    }
}package com.jarvis.app

import android.Manifest
import android.accessibilityservice.AccessibilityService
import android.content.Context
import android.content.Intent
import android.hardware.camera2.CameraManager
import android.net.Uri
import android.os.Bundle
import android.speech.RecognitionListener
import android.speech.RecognizerIntent
import android.speech.SpeechRecognizer
import android.speech.tts.TextToSpeech
import android.view.Gravity
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import okhttp3.*
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.RequestBody.Companion.toRequestBody
import org.json.JSONArray
import org.json.JSONObject
import java.util.Locale
import kotlin.concurrent.thread

class MainActivity : AppCompatActivity() {
    private val API_KEY = "YOUR_API_KEY"
    private lateinit var tts: TextToSpeech
    private lateinit var rec: SpeechRecognizer
    private lateinit var out: TextView
    private val http = OkHttpClient()
    private val history = JSONArray()

    override fun onCreate(b: Bundle?) {
        super.onCreate(b)
        out = TextView(this).apply { textSize = 18f; text = "JARVIS तैयार है" }
        val btn = Button(this).apply { text = "🎤 बोलिए"; setOnClickListener { listen() } }
        setContentView(LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER; setPadding(48, 48, 48, 48)
            addView(out); addView(btn)
        })
        ActivityCompat.requestPermissions(this, arrayOf(
            Manifest.permission.RECORD_AUDIO, Manifest.permission.CALL_PHONE,
            Manifest.permission.CAMERA), 1)

        tts = TextToSpeech(this) { tts.language = Locale("hi", "IN") }
        rec = SpeechRecognizer.createSpeechRecognizer(this)
        rec.setRecognitionListener(object : RecognitionListener {
            override fun onResults(r: Bundle?) {
                val t = r?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull() ?: return
                out.text = "आप: $t"; handle(t)
            }
            override fun onReadyForSpeech(p: Bundle?) {}
            override fun onBeginningOfSpeech() {}
            override fun onRmsChanged(v: Float) {}
            override fun onBufferReceived(b: ByteArray?) {}
            override fun onEndOfSpeech() {}
            override fun onError(e: Int) { out.text = "समझ नहीं आया, दोबारा बोलिए" }
            override fun onPartialResults(p: Bundle?) {}
            override fun onEvent(t: Int, p: Bundle?) {}
        })
    }

    private fun listen() {
        rec.startListening(Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, "hi-IN")
        })
    }

    private fun say(s: String) {
        runOnUiThread { out.text = "JARVIS: $s" }
        tts.speak(s, TextToSpeech.QUEUE_FLUSH, null, null)
    }

    private fun handle(t: String) {
        val c = t.lowercase()
        val svc = JarvisService.instance
        fun has(vararg k: String) = k.any { c.contains(it) }
        when {
            has("torch on", "flashlight on", "टॉर्च चालू", "टॉर्च जलाओ") -> { torch(true); say("टॉर्च चालू") }
            has("torch off", "flashlight off", "टॉर्च बंद") -> { torch(false); say("टॉर्च बंद") }
            has("home", "होम") -> svc?.performGlobalAction(AccessibilityService.GLOBAL_ACTION_HOME) ?: say("Accessibility ON करें")
            has("back", "पीछे") -> svc?.performGlobalAction(AccessibilityService.GLOBAL_ACTION_BACK)
            has("notification", "नोटिफिकेशन") -> svc?.performGlobalAction(AccessibilityService.GLOBAL_ACTION_NOTIFICATIONS)
            has("screenshot", "स्क्रीनशॉट") -> svc?.performGlobalAction(AccessibilityService.GLOBAL_ACTION_TAKE_SCREENSHOT)
            has("recent", "रीसेंट") -> svc?.performGlobalAction(AccessibilityService.GLOBAL_ACTION_RECENTS)
            has("call", "कॉल") -> {
                val num = c.filter { it.isDigit() }
                if (num.isNotEmpty()) {
                    startActivity(Intent(Intent.ACTION_CALL, Uri.parse("tel:$num")))
                    say("कॉल लगा रहा हूँ")
                } else say("नंबर बोलिए")
            }
            has("search", "सर्च", "गूगल") -> {
                val q = c.replace(Regex("search|सर्च|गूगल"), "").trim()
                startActivity(Intent(Intent.ACTION_VIEW, Uri.parse("https://www.google.com/search?q=" + Uri.encode(q))))
            }
            has("open", "खोल", "kholo", "ओपन") -> {
                val name = c.replace(Regex("open|खोलो|खोल|kholo|ओपन"), "").trim()
                if (openApp(name)) say("$name खोल रहा हूँ") else say("$name ऐप नहीं मिला")
            }
            has("click", "दबाओ", "dabao") -> {
                val name = c.replace(Regex("click|दबाओ|dabao"), "").trim()
                say(if (svc?.clickText(name) == true) "हो गया" else "बटन नहीं मिला")
            }
            else -> askClaude(t)
        }
    }

    private fun openApp(name: String): Boolean {
        val pm = packageManager
        val app = pm.getInstalledApplications(0).firstOrNull {
            pm.getApplicationLabel(it).toString().lowercase().contains(name)
        } ?: return false
        val i = pm.getLaunchIntentForPackage(app.packageName) ?: return false
        startActivity(i); return true
    }

    private fun torch(on: Boolean) {
        val cm = getSystemService(Context.CAMERA_SERVICE) as CameraManager
        cm.setTorchMode(cm.cameraIdList[0], on)
    }

    private fun askClaude(q: String) {
        history.put(JSONObject().put("role", "user").put("content", q))
        val body = JSONObject()
            .put("model", "claude-sonnet-5-5")
            .put("max_tokens", 500)
            .put("system", "तुम JARVIS हो, मेरे फोन का असिस्टेंट। हिंदी में छोटा और सीधा जवाब दो।")
            .put("messages", history).toString()
            .toRequestBody("application/json".toMediaType())
        val req = Request.Builder().url("https://api.anthropic.com/v1/messages")
            .header("x-api-key", API_KEY)
            .header("anthropic-version", "2023-06-01")
            .post(body).build()
        thread {
            try {
                val res = JSONObject(http.newCall(req).execute().body!!.string())
                val ans = res.getJSONArray("content").getJSONObject(0).getString("text")
                history.put(JSONObject().put("role", "assistant").put("content", ans))
                say(ans)
            } catch (e: Exception) { say("इंटरनेट या API Key में दिक्कत है") }
        }
    }
}
