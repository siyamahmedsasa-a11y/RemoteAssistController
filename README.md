// RemoteAssist Android Host (Java) - Project Starter Files
// -----------------------------------------------------------------
// Files included (in one document):
// 1) app/build.gradle (module)
// 2) AndroidManifest.xml
// 3) MainActivity.java
// 4) ScreenShareService.java
// 5) FirebaseSignalingHelper.java (stub)
// 6) README.md
// -----------------------------------------------------------------

--- app/build.gradle ---
apply plugin: 'com.android.application'

android {
    compileSdkVersion 34

    defaultConfig {
        applicationId "com.example.remoteassist.host"
        minSdkVersion 21
        targetSdkVersion 34
        versionCode 1
        versionName "1.0"
        multiDexEnabled true
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }
}

dependencies {
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation 'com.google.firebase:firebase-database:20.2.0' // signaling helper (optional)
    implementation 'com.google.firebase:firebase-firestore:24.7.1'
    implementation 'com.google.code.gson:gson:2.10.1'
    implementation 'org.webrtc:google-webrtc:1.0.32006' // community artifact; you may need to obtain proper libwebrtc
}

// Add google services plugin in project-level build.gradle if using Firebase

--- AndroidManifest.xml ---
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.remoteassist.host">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <application
        android:allowBackup="true"
        android:label="RemoteAssist Host"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true">

        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".ScreenShareService"
            android:foregroundServiceType="mediaProjection"
            android:exported="false" />

    </application>
</manifest>

--- app/src/main/java/com/example/remoteassist/host/MainActivity.java ---
package com.example.remoteassist.host;

import android.app.Activity;
import android.content.Intent;
import android.media.projection.MediaProjectionManager;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    private static final int REQUEST_CODE_SCREEN_CAPTURE = 1001;
    private MediaProjectionManager projectionManager;
    private Button btnStart, btnStop;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btnStart = findViewById(R.id.btnStart);
        btnStop = findViewById(R.id.btnStop);

        projectionManager = (MediaProjectionManager) getSystemService(MEDIA_PROJECTION_SERVICE);

        btnStart.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startScreenCapture();
            }
        });

        btnStop.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                stopService(new Intent(MainActivity.this, ScreenShareService.class));
                Toast.makeText(MainActivity.this, "Stopped sharing", Toast.LENGTH_SHORT).show();
            }
        });
    }

    private void startScreenCapture() {
        Intent intent = projectionManager.createScreenCaptureIntent();
        startActivityForResult(intent, REQUEST_CODE_SCREEN_CAPTURE);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_CODE_SCREEN_CAPTURE) {
            if (resultCode == Activity.RESULT_OK && data != null) {
                Intent serviceIntent = new Intent(this, ScreenShareService.class);
                serviceIntent.putExtra("resultCode", resultCode);
                serviceIntent.putExtra("resultData", data);
                startForegroundService(serviceIntent);
                Toast.makeText(this, "Started sharing", Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(this, "Screen capture permission denied", Toast.LENGTH_SHORT).show();
            }
        }
    }
}

--- app/src/main/java/com/example/remoteassist/host/ScreenShareService.java ---
package com.example.remoteassist.host;

import android.app.Notification;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.app.PendingIntent;
import android.app.Service;
import android.content.Intent;
import android.media.projection.MediaProjection;
import android.media.projection.MediaProjectionManager;
import android.os.Build;
import android.os.IBinder;
import android.util.Log;

import androidx.annotation.Nullable;
import androidx.core.app.NotificationCompat;

// NOTE: This is a skeleton service. WebRTC initialization and streaming logic should be added.
public class ScreenShareService extends Service {

    private static final String TAG = "ScreenShareService";
    private static final String CHANNEL_ID = "remoteassist_share";

    private MediaProjection mediaProjection;

    @Override
    public void onCreate() {
        super.onCreate();
        createNotificationChannel();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        int resultCode = intent.getIntExtra("resultCode", -1);
        Intent resultData = intent.getParcelableExtra("resultData");

        if (resultCode != -1 && resultData != null) {
            MediaProjectionManager mgr = (MediaProjectionManager) getSystemService(MEDIA_PROJECTION_SERVICE);
            mediaProjection = mgr.getMediaProjection(resultCode, resultData);
            // TODO: Initialize WebRTC PeerConnectionFactory and start capturing with ScreenCapturerAndroid
            Log.i(TAG, "MediaProjection obtained, TODO: start WebRTC capture");
        } else {
            Log.w(TAG, "No media projection data provided");
        }

        Intent activityIntent = new Intent(this, MainActivity.class);
        PendingIntent pi = PendingIntent.getActivity(this, 0, activityIntent, PendingIntent.FLAG_IMMUTABLE);

        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setContentTitle("RemoteAssist: Sharing screen")
                .setContentText("Tap to open")
                .setSmallIcon(android.R.drawable.ic_media_play)
                .setContentIntent(pi)
                .setOngoing(true)
                .build();

        startForeground(1, notification);

        return START_STICKY;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mediaProjection != null) {
            mediaProjection.stop();
            mediaProjection = null;
        }
        Log.i(TAG, "Service destroyed");
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(CHANNEL_ID, "Screen Sharing", NotificationManager.IMPORTANCE_LOW);
            NotificationManager nm = getSystemService(NotificationManager.class);
            if (nm != null) nm.createNotificationChannel(channel);
        }
    }
}

--- app/src/main/java/com/example/remoteassist/host/FirebaseSignalingHelper.java ---
package com.example.remoteassist.host;

// Minimal stub for signaling via Firebase (you need to fill in actual Firestore or Realtime DB code).
public class FirebaseSignalingHelper {
    // TODO: implement offer/answer/ice exchange using Firestore or Realtime DB
}

--- README.md ---
# RemoteAssist Host (Java) - Starter

এই প্রজেক্টে Host অ্যাপের বেসিক স্ট্রাকচার আছে — ScreenCapture permission flow এবং foreground service skeleton।

## দ্রুত শুরু
1. Android Studio দিয়ে নতুন project টাওয়ান; উপরের ফাইলগুলো অনুকরণ করে রাখো।
2. Gradle dependencies: `org.webrtc:google-webrtc` কে তুমি নিজের বিল্ড সার্ভারে বা maven জায়গা থেকে যোগ করতে পারো; অনেক প্রকল্পকেই prebuilts প্রয়োজন।
3. Firebase signaling ব্যবহার করতে হলে project-level configuration (google-services.json) যোগ করো এবং Firebase SDK plugin enable করো।
4. Run: দুইটি ফোন দরকার — Host ওপেন করে Start -> system dialog -> Start sharing; Controller কোড পরে দিব।

## পরবর্তী ধাপ
- WebRTC PeerConnection init (PeerConnectionFactory), ScreenCapturerAndroid ব্যবহার করে frame capture
- signaling (Firebase or Node.js WebSocket)
- Controller অ্যাপ - receive stream, show, send datachannel input

---
// End of document
