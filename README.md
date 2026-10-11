pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement { repositories { google(); mavenCentral() } }
rootProject.name = "UrduTranscriptPro"
include(":app") plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "2.0.20" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.20" apply false
} android.useAndroidX=true
org.gradle.jvmargs=-Xmx3g distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.9-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}
android {
    namespace = "com.urdutranscript.pro"
    compileSdk = 34
    ndkVersion = "26.1.10909125"
    defaultConfig {
        applicationId = "com.urdutranscript.pro"
        minSdk = 26; targetSdk = 34
        versionCode = 1; versionName = "0.1"
        ndk { abiFilters += listOf("arm64-v8a") }
        externalNativeBuild { cmake {
            arguments += listOf("-DCMAKE_BUILD_TYPE=Release", "-DWHISPER_BUILD_TESTS=OFF",
                "-DWHISPER_BUILD_EXAMPLES=OFF", "-DGGML_NATIVE=OFF")
        } }
    }
    externalNativeBuild { cmake { path = file("src/main/cpp/CMakeLists.txt"); version = "3.22.1" } }
    buildTypes { release { isMinifyEnabled = false } }
    compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
    buildFeatures { compose = true }
}
dependencies {
    implementation(platform("androidx.compose:compose-bom:2024.09.00"))
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.activity:activity-compose:1.9.2")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.media3:media3-exoplayer:1.4.1")
    implementation("androidx.media3:media3-ui:1.4.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
} <?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application android:label="Urdu Video Transcript Pro" android:supportsRtl="true"
        android:theme="@android:style/Theme.Material.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true"
            android:configChanges="orientation|screenSize|keyboardHidden">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest> cmake_minimum_required(VERSION 3.22.1)
project(whisperjni)
set(CMAKE_CXX_STANDARD 17)
set(BUILD_SHARED_LIBS OFF)
add_subdirectory(whisper.cpp whisper_build)
add_library(whisperjni SHARED jni.cpp)
target_include_directories(whisperjni PRIVATE whisper.cpp/include whisper.cpp/ggml/include)
target_link_libraries(whisperjni whisper log android) #include <jni.h>
#include <string>
#include <vector>
#include "whisper.h"

static std::string sanitize(const char* s) {
    std::string out; const unsigned char* p = (const unsigned char*)s;
    while (*p) {
        int n = (*p < 0x80) ? 1 : ((*p >> 5) == 6) ? 2 : ((*p >> 4) == 14) ? 3 : ((*p >> 3) == 30) ? 4 : 0;
        bool ok = n > 0;
        for (int i = 1; ok && i < n; i++) if ((p[i] & 0xC0) != 0x80) ok = false;
        if (ok) { out.append((const char*)p, n); p += n; } else p++;
    }
    return out;
}

extern "C" {
JNIEXPORT jlong JNICALL Java_com_urdutranscript_pro_WhisperLib_init(JNIEnv* env, jobject, jstring path) {
    const char* p = env->GetStringUTFChars(path, nullptr);
    whisper_context_params cp = whisper_context_default_params();
    cp.use_gpu = false;
    whisper_context* c = whisper_init_from_file_with_params(p, cp);
    env->ReleaseStringUTFChars(path, p);
    return (jlong)c;
}
JNIEXPORT void JNICALL Java_com_urdutranscript_pro_WhisperLib_free(JNIEnv*, jobject, jlong ctx) {
    if (ctx) whisper_free((whisper_context*)ctx);
}
JNIEXPORT jobjectArray JNICALL Java_com_urdutranscript_pro_WhisperLib_transcribe(
        JNIEnv* env, jobject, jlong ctx, jfloatArray samples, jint threads) {
    auto* c = (whisper_context*)ctx;
    jsize n = env->GetArrayLength(samples);
    jfloat* data = env->GetFloatArrayElements(samples, nullptr);
    whisper_full_params wp = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
    wp.language = "ur"; wp.translate = false; wp.detect_language = false;
    wp.n_threads = threads; wp.no_context = true;
    wp.print_progress = wp.print_realtime = wp.print_timestamps = wp.print_special = false;
    int rc = whisper_full(c, wp, data, n);
    env->ReleaseFloatArrayElements(samples, data, JNI_ABORT);
    int count = rc == 0 ? whisper_full_n_segments(c) : 0;
    jclass sc = env->FindClass("java/lang/String");
    jobjectArray arr = env->NewObjectArray(count, sc, nullptr);
    for (int i = 0; i < count; i++) {
        std::string s = std::to_string(whisper_full_get_segment_t0(c, i)) + "|" +
                        sanitize(whisper_full_get_segment_text(c, i));
        env->SetObjectArrayElement(arr, i, env->NewStringUTF(s.c_str()));
    }
    return arr;
}
} package com.urdutranscript.pro

object WhisperLib {
    init { System.loadLibrary("whisperjni") }
    external fun init(modelPath: String): Long
    external fun transcribe(ctx: Long, samples: FloatArray, threads: Int): Array<String>
    external fun free(ctx: Long)
} package com.urdutranscript.pro

import android.content.Context
import android.media.MediaCodec
import android.media.MediaExtractor
import android.media.MediaFormat
import android.net.Uri
import kotlinx.coroutines.currentCoroutineContext
import kotlinx.coroutines.ensureActive
import java.nio.ByteOrder

class NoAudioTrackException : Exception()

object AudioExtractor {
    private const val RATE = 16000
    private const val CHUNK = RATE * 30

    suspend fun decodeChunks(ctx: Context, uri: Uri, onChunk: suspend (FloatArray, Long) -> Unit) {
        val ex = MediaExtractor()
        ex.setDataSource(ctx, uri, null)
        val track = (0 until ex.trackCount).firstOrNull {
            ex.getTrackFormat(it).getString(MediaFormat.KEY_MIME)?.startsWith("audio/") == true
        }
        if (track == null) { ex.release(); throw NoAudioTrackException() }
        ex.selectTrack(track)
        val fmt = ex.getTrackFormat(track)
        val codec = MediaCodec.createDecoderByType(fmt.getString(MediaFormat.KEY_MIME)!!)
        try {
            codec.configure(fmt, null, null, 0); codec.start()
            var srcRate = fmt.getInteger(MediaFormat.KEY_SAMPLE_RATE)
            var ch = fmt.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
            var step = srcRate.toDouble() / RATE
            var next = 0.0; var n = -1L; var prev = 0f
            val buf = FloatArray(CHUNK); var filled = 0; var chunkIdx = 0L
            val info = MediaCodec.BufferInfo()
            var inDone = false; var outDone = false

            suspend fun push(v: Float) {
                buf[filled++] = v
                if (filled == CHUNK) { onChunk(buf.copyOf(), chunkIdx * 30_000L); chunkIdx++; filled = 0 }
            }
            while (!outDone) {
                currentCoroutineContext().ensureActive()
                if (!inDone) {
                    val i = codec.dequeueInputBuffer(5000)
                    if (i >= 0) {
                        val size = ex.readSampleData(codec.getInputBuffer(i)!!, 0)
                        if (size < 0) {
                            codec.queueInputBuffer(i, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM); inDone = true
                        } else { codec.queueInputBuffer(i, 0, size, ex.sampleTime, 0); ex.advance() }
                    }
                }
                val o = codec.dequeueOutputBuffer(info, 5000)
                if (o >= 0) {
                    val sb = codec.getOutputBuffer(o)!!.order(ByteOrder.nativeOrder()).asShortBuffer()
                    val frames = sb.remaining() / ch
                    for (f in 0 until frames) {
                        var sum = 0f
                        for (c in 0 until ch) sum += sb.get(f * ch + c) / 32768f
                        val s = sum / ch; n++
                        while (next <= n) {
                            push(prev + (s - prev) * (next - (n - 1)).toFloat()); next += step
                        }
                        prev = s
                    }
                    codec.releaseOutputBuffer(o, false)
                    if (info.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) outDone = true
                } else if (o == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                    val f = codec.outputFormat
                    srcRate = f.getInteger(MediaFormat.KEY_SAMPLE_RATE)
                    ch = f.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
                    step = srcRate.toDouble() / RATE
                }
            }
            if (filled > RATE / 2) onChunk(buf.copyOf(maxOf(filled, RATE)), chunkIdx * 30_000L)
        } finally {
            try { codec.stop() } catch (_: Exception) {}
            codec.release(); ex.release()
        }
    }
} package com.urdutranscript.pro

import android.content.Context
import android.media.MediaCodec
import android.media.MediaExtractor
import android.media.MediaFormat
import android.net.Uri
import kotlinx.coroutines.currentCoroutineContext
import kotlinx.coroutines.ensureActive
import java.nio.ByteOrder

class NoAudioTrackException : Exception()

object AudioExtractor {
    private const val RATE = 16000
    private const val CHUNK = RATE * 30

    suspend fun decodeChunks(ctx: Context, uri: Uri, onChunk: suspend (FloatArray, Long) -> Unit) {
        val ex = MediaExtractor()
        ex.setDataSource(ctx, uri, null)
        val track = (0 until ex.trackCount).firstOrNull {
            ex.getTrackFormat(it).getString(MediaFormat.KEY_MIME)?.startsWith("audio/") == true
        }
        if (track == null) { ex.release(); throw NoAudioTrackException() }
        ex.selectTrack(track)
        val fmt = ex.getTrackFormat(track)
        val codec = MediaCodec.createDecoderByType(fmt.getString(MediaFormat.KEY_MIME)!!)
        try {
            codec.configure(fmt, null, null, 0); codec.start()
            var srcRate = fmt.getInteger(MediaFormat.KEY_SAMPLE_RATE)
            var ch = fmt.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
            var step = srcRate.toDouble() / RATE
            var next = 0.0; var n = -1L; var prev = 0f
            val buf = FloatArray(CHUNK); var filled = 0; var chunkIdx = 0L
            val info = MediaCodec.BufferInfo()
            var inDone = false; var outDone = false

            suspend fun push(v: Float) {
                buf[filled++] = v
                if (filled == CHUNK) { onChunk(buf.copyOf(), chunkIdx * 30_000L); chunkIdx++; filled = 0 }
            }
            while (!outDone) {
                currentCoroutineContext().ensureActive()
                if (!inDone) {
                    val i = codec.dequeueInputBuffer(5000)
                    if (i >= 0) {
                        val size = ex.readSampleData(codec.getInputBuffer(i)!!, 0)
                        if (size < 0) {
                            codec.queueInputBuffer(i, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM); inDone = true
                        } else { codec.queueInputBuffer(i, 0, size, ex.sampleTime, 0); ex.advance() }
                    }
                }
                val o = codec.dequeueOutputBuffer(info, 5000)
                if (o >= 0) {
                    val sb = codec.getOutputBuffer(o)!!.order(ByteOrder.nativeOrder()).asShortBuffer()
                    val frames = sb.remaining() / ch
                    for (f in 0 until frames) {
                        var sum = 0f
                        for (c in 0 until ch) sum += sb.get(f * ch + c) / 32768f
                        val s = sum / ch; n++
                        while (next <= n) {
                            push(prev + (s - prev) * (next - (n - 1)).toFloat()); next += step
                        }
                        prev = s
                    }
                    codec.releaseOutputBuffer(o, false)
                    if (info.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) outDone = true
                } else if (o == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                    val f = codec.outputFormat
                    srcRate = f.getInteger(MediaFormat.KEY_SAMPLE_RATE)
                    ch = f.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
                    step = srcRate.toDouble() / RATE
                }
            }
            if (filled > RATE / 2) onChunk(buf.copyOf(maxOf(filled, RATE)), chunkIdx * 30_000L)
        } finally {
            try { codec.stop() } catch (_: Exception) {}
            codec.release(); ex.release()
        }
    }
} package com.urdutranscript.pro

import android.content.Context
import android.media.MediaCodec
import android.media.MediaExtractor
import android.media.MediaFormat
import android.net.Uri
import kotlinx.coroutines.currentCoroutineContext
import kotlinx.coroutines.ensureActive
import java.nio.ByteOrder

class NoAudioTrackException : Exception()

object AudioExtractor {
    private const val RATE = 16000
    private const val CHUNK = RATE * 30

    suspend fun decodeChunks(ctx: Context, uri: Uri, onChunk: suspend (FloatArray, Long) -> Unit) {
        val ex = MediaExtractor()
        ex.setDataSource(ctx, uri, null)
        val track = (0 until ex.trackCount).firstOrNull {
            ex.getTrackFormat(it).getString(MediaFormat.KEY_MIME)?.startsWith("audio/") == true
        }
        if (track == null) { ex.release(); throw NoAudioTrackException() }
        ex.selectTrack(track)
        val fmt = ex.getTrackFormat(track)
        val codec = MediaCodec.createDecoderByType(fmt.getString(MediaFormat.KEY_MIME)!!)
        try {
            codec.configure(fmt, null, null, 0); codec.start()
            var srcRate = fmt.getInteger(MediaFormat.KEY_SAMPLE_RATE)
            var ch = fmt.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
            var step = srcRate.toDouble() / RATE
            var next = 0.0; var n = -1L; var prev = 0f
            val buf = FloatArray(CHUNK); var filled = 0; var chunkIdx = 0L
            val info = MediaCodec.BufferInfo()
            var inDone = false; var outDone = false

            suspend fun push(v: Float) {
                buf[filled++] = v
                if (filled == CHUNK) { onChunk(buf.copyOf(), chunkIdx * 30_000L); chunkIdx++; filled = 0 }
            }
            while (!outDone) {
                currentCoroutineContext().ensureActive()
                if (!inDone) {
                    val i = codec.dequeueInputBuffer(5000)
                    if (i >= 0) {
                        val size = ex.readSampleData(codec.getInputBuffer(i)!!, 0)
                        if (size < 0) {
                            codec.queueInputBuffer(i, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM); inDone = true
                        } else { codec.queueInputBuffer(i, 0, size, ex.sampleTime, 0); ex.advance() }
                    }
                }
                val o = codec.dequeueOutputBuffer(info, 5000)
                if (o >= 0) {
                    val sb = codec.getOutputBuffer(o)!!.order(ByteOrder.nativeOrder()).asShortBuffer()
                    val frames = sb.remaining() / ch
                    for (f in 0 until frames) {
                        var sum = 0f
                        for (c in 0 until ch) sum += sb.get(f * ch + c) / 32768f
                        val s = sum / ch; n++
                        while (next <= n) {
                            push(prev + (s - prev) * (next - (n - 1)).toFloat()); next += step
                        }
                        prev = s
                    }
                    codec.releaseOutputBuffer(o, false)
                    if (info.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) outDone = true
                } else if (o == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                    val f = codec.outputFormat
                    srcRate = f.getInteger(MediaFormat.KEY_SAMPLE_RATE)
                    ch = f.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
                    step = srcRate.toDouble() / RATE
                }
            }
            if (filled > RATE / 2) onChunk(buf.copyOf(maxOf(filled, RATE)), chunkIdx * 30_000L)
        } finally {
            try { codec.stop() } catch (_: Exception) {}
            codec.release(); ex.release()
        }
    }
}۔ package com.urdutranscript.pro

import android.content.Context
import android.media.MediaCodec
import android.media.MediaExtractor
import android.media.MediaFormat
import android.net.Uri
import kotlinx.coroutines.currentCoroutineContext
import kotlinx.coroutines.ensureActive
import java.nio.ByteOrder

class NoAudioTrackException : Exception()

object AudioExtractor {
    private const val RATE = 16000
    private const val CHUNK = RATE * 30

    suspend fun decodeChunks(ctx: Context, uri: Uri, onChunk: suspend (FloatArray, Long) -> Unit) {
        val ex = MediaExtractor()
        ex.setDataSource(ctx, uri, null)
        val track = (0 until ex.trackCount).firstOrNull {
            ex.getTrackFormat(it).getString(MediaFormat.KEY_MIME)?.startsWith("audio/") == true
        }
        if (track == null) { ex.release(); throw NoAudioTrackException() }
        ex.selectTrack(track)
        val fmt = ex.getTrackFormat(track)
        val codec = MediaCodec.createDecoderByType(fmt.getString(MediaFormat.KEY_MIME)!!)
        try {
            codec.configure(fmt, null, null, 0); codec.start()
            var srcRate = fmt.getInteger(MediaFormat.KEY_SAMPLE_RATE)
            var ch = fmt.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
            var step = srcRate.toDouble() / RATE
            var next = 0.0; var n = -1L; var prev = 0f
            val buf = FloatArray(CHUNK); var filled = 0; var chunkIdx = 0L
            val info = MediaCodec.BufferInfo()
            var inDone = false; var outDone = false

            suspend fun push(v: Float) {
                buf[filled++] = v
                if (filled == CHUNK) { onChunk(buf.copyOf(), chunkIdx * 30_000L); chunkIdx++; filled = 0 }
            }
            while (!outDone) {
                currentCoroutineContext().ensureActive()
                if (!inDone) {
                    val i = codec.dequeueInputBuffer(5000)
                    if (i >= 0) {
                        val size = ex.readSampleData(codec.getInputBuffer(i)!!, 0)
                        if (size < 0) {
                            codec.queueInputBuffer(i, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM); inDone = true
                        } else { codec.queueInputBuffer(i, 0, size, ex.sampleTime, 0); ex.advance() }
                    }
                }
                val o = codec.dequeueOutputBuffer(info, 5000)
                if (o >= 0) {
                    val sb = codec.getOutputBuffer(o)!!.order(ByteOrder.nativeOrder()).asShortBuffer()
                    val frames = sb.remaining() / ch
                    for (f in 0 until frames) {
                        var sum = 0f
                        for (c in 0 until ch) sum += sb.get(f * ch + c) / 32768f
                        val s = sum / ch; n++
                        while (next <= n) {
                            push(prev + (s - prev) * (next - (n - 1)).toFloat()); next += step
                        }
                        prev = s
                    }
                    codec.releaseOutputBuffer(o, false)
                    if (info.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) outDone = true
                } else if (o == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
                    val f = codec.outputFormat
                    srcRate = f.getInteger(MediaFormat.KEY_SAMPLE_RATE)
                    ch = f.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
                    step = srcRate.toDouble() / RATE
                }
            }
            if (filled > RATE / 2) onChunk(buf.copyOf(maxOf(filled, RATE)), chunkIdx * 30_000L)
        } finally {
            try { codec.stop() } catch (_: Exception) {}
            codec.release(); ex.release()
        }
    }
} package com.urdutranscript.pro

import android.app.Application
import android.content.ClipData
import android.content.ClipboardManager
import android.content.Context
import android.media.MediaMetadataRetriever
import android.net.Uri
import android.os.Bundle
import android.provider.OpenableColumns
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.itemsIndexed
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalLayoutDirection
import androidx.compose.ui.unit.LayoutDirection
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.compose.ui.viewinterop.AndroidView
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.media3.common.MediaItem
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.ui.PlayerView
import kotlinx.coroutines.*
import java.io.File

data class Seg(val startMs: Long, val text: String)

class MainVM(app: Application) : AndroidViewModel(app) {
    private val modelFile = File(app.filesDir, "model.bin")
    var videoUri by mutableStateOf<Uri?>(null)
    var videoName by mutableStateOf("")
    var durationMs by mutableLongStateOf(0L)
    var status by mutableStateOf("ویڈیو منتخب کریں")
    var progress by mutableFloatStateOf(0f)
    var modelReady by mutableStateOf(modelFile.exists())
    val segments = mutableStateListOf<Seg>()
    private var job: Job? = null
    val running get() = job?.isActive == true

    fun importVideo(uri: Uri) {
        val app = getApplication<Application>()
        videoUri = uri; segments.clear(); progress = 0f; status = "ویڈیو تیار ہے"
        app.contentResolver.query(uri, null, null, null, null)?.use {
            if (it.moveToFirst()) videoName = it.getString(it.getColumnIndexOrThrow(OpenableColumns.DISPLAY_NAME))
        }
        runCatching {
            val r = MediaMetadataRetriever(); r.setDataSource(app, uri)
            durationMs = r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_DURATION)?.toLong() ?: 0L
            r.release()
        }
    }

    fun importModel(uri: Uri) = viewModelScope.launch(Dispatchers.IO) {
        withContext(Dispatchers.Main) { status = "ماڈل کاپی ہو رہا ہے…" }
        try {
            val tmp = File(modelFile.path + ".tmp")
            getApplication<Application>().contentResolver.openInputStream(uri)!!.use { i -> tmp.outputStream().use { i.copyTo(it) } }
            tmp.renameTo(modelFile)
            withContext(Dispatchers.Main) { modelReady = true; status = "ماڈل تیار ہے" }
        } catch (e: Exception) {
            withContext(Dispatchers.Main) { status = "ماڈل کاپی کرنے میں مسئلہ (جگہ چیک کریں)" }
        }
    }

    fun stop() { job?.cancel() }

    fun start() {
        val uri = videoUri ?: return
        if (!modelReady) { status = "پہلے ماڈل فائل منتخب کریں"; return }
        if (running) return
        segments.clear(); progress = 0f
        val threads = Runtime.getRuntime().availableProcessors().coerceIn(2, 4)
        job = viewModelScope.launch(Dispatchers.Default) {
            var ctx = 0L
            suspend fun say(s: String) = withContext(Dispatchers.Main + NonCancellable) { status = s }
            try {
                say("آواز کا تجزیہ جاری ہے")
                ctx = WhisperLib.init(modelFile.absolutePath)
                check(ctx != 0L) { "model load failed" }
                AudioExtractor.decodeChunks(getApplication(), uri) { chunk, startMs ->
                    say("اردو تحریر تیار ہو رہی ہے")
                    val res = WhisperLib.transcribe(ctx, chunk, threads)
                    withContext(Dispatchers.Main) {
                        for (r in res) {
                            val i = r.indexOf('|')
                            val txt = r.substring(i + 1).trim()
                            if (txt.isNotEmpty() && txt != segments.lastOrNull()?.text)
                                segments.add(Seg(startMs + r.substring(0, i).toLong() * 10, txt))
                        }
                        if (durationMs > 0) progress = ((startMs + 30_000f) / durationMs).coerceAtMost(1f)
                    }
                }
                say("ٹرانسکرپشن مکمل ہو گئی"); withContext(Dispatchers.Main) { progress = 1f }
            } catch (e: CancellationException) { say("ٹرانسکرپشن روک دی گئی"); throw e
            } catch (e: NoAudioTrackException) { say("اس ویڈیو میں آواز موجود نہیں")
            } catch (e: Throwable) { say("پراسیسنگ میں مسئلہ پیش آیا: ${e.message ?: ""}")
            } finally { if (ctx != 0L) WhisperLib.free(ctx) }
        }
    }
}

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { App() }
    }
}

private val Charcoal = Color(0xFF1C1F1E)
private val CardBg = Color(0xFF262B29)
private val Green = Color(0xFF1B7A5A)

@Composable
fun App(vm: MainVM = viewModel()) {
    val ctx = LocalContext.current
    val videoPicker = rememberLauncherForActivityResult(ActivityResultContracts.OpenDocument()) { it?.let(vm::importVideo) }
    val modelPicker = rememberLauncherForActivityResult(ActivityResultContracts.OpenDocument()) { it?.let(vm::importModel) }
    val player = remember { ExoPlayer.Builder(ctx).build() }
    DisposableEffect(Unit) { onDispose { player.release() } }
    LaunchedEffect(vm.videoUri) { vm.videoUri?.let { player.setMediaItem(MediaItem.fromUri(it)); player.prepare() } }

    CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
        MaterialTheme(colorScheme = darkColorScheme(primary = Green, background = Charcoal, surface = CardBg)) {
            Surface(Modifier.fillMaxSize(), color = Charcoal) {
                Column(Modifier.systemBarsPadding().padding(12.dp), verticalArrangement = Arrangement.spacedBy(10.dp)) {
                    AndroidView(
                        factory = { PlayerView(it).apply { this.player = player } },
                        modifier = Modifier.fillMaxWidth().height(210.dp)
                    )
                    if (vm.videoName.isNotEmpty())
                        Text("${vm.videoName} • ${vm.durationMs / 60000}:${"%02d".format(vm.durationMs / 1000 % 60)}", color = Color.White, fontSize = 14.sp)
                    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        Button(onClick = { videoPicker.launch(arrayOf("video/*")) }, Modifier.weight(1f)) { Text("ویڈیو منتخب کریں") }
                        OutlinedButton(onClick = { modelPicker.launch(arrayOf("*/*")) }, Modifier.weight(1f)) {
                            Text(if (vm.modelReady) "ماڈل بدلیں" else "ماڈل فائل منتخب کریں")
                        }
                    }
                    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        Button(onClick = { vm.start() }, Modifier.weight(1f), enabled = vm.videoUri != null && !vm.running) { Text("ٹرانسکرپشن شروع کریں") }
                        OutlinedButton(onClick = { vm.stop() }, Modifier.weight(1f), enabled = vm.running) { Text("روکیں") }
                    }
                    Text(vm.status, color = Color(0xFFB7E4C7), fontSize = 14.sp)
                    LinearProgressIndicator(progress = { vm.progress }, Modifier.fillMaxWidth())
                    LazyColumn(Modifier.weight(1f).fillMaxWidth(), verticalArrangement = Arrangement.spacedBy(8.dp)) {
                        itemsIndexed(vm.segments) { _, seg ->
                            Card(Modifier.fillMaxWidth().clickable { player.seekTo(seg.startMs) },
                                shape = RoundedCornerShape(14.dp), colors = CardDefaults.cardColors(containerColor = CardBg)) {
                                Text(seg.text, Modifier.padding(14.dp), color = Color(0xFFF5F5F0), fontSize = 20.sp, lineHeight = 34.sp)
                            }
                        }
                    }
                    OutlinedButton(onClick = {
                        val cm = ctx.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
                        cm.setPrimaryClip(ClipData.newPlainText("transcript", vm.segments.joinToString("\n") { it.text }))
                    }, Modifier.fillMaxWidth(), enabled = vm.segments.isNotEmpty()) { Text("تمام متن کاپی کریں") }
                }
            }
        }
    }
}name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - name: Fetch whisper.cpp
        run: git clone --depth 1 --branch v1.7.2 https://github.com/ggerganov/whisper.cpp app/src/main/cpp/whisper.cpp
      - name: Install NDK and CMake
        run: yes | $ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager "ndk;26.1.10909125" "cmake;3.22.1" "platforms;android-34"
      - uses: gradle/actions/setup-gradle@v4
        with: { gradle-version: "8.9" }
      - run: gradle :app:assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: UrduTranscriptPro-apk
          path: app/build/outputs/apk/debug/app-debug.apk
