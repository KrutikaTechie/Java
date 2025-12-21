<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.privasentry">

    <!-- Read access (safe for college project & Android 10+) -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>

    <application
        android:allowBackup="true"
        android:label="PrivaSentry"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.Light.DarkActionBar">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

    </application>
</manifest>    


//second code
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="#F8F9FA">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="PrivaSentry: Data Guard"
        android:textSize="24sp"
        android:textStyle="bold"
        android:textColor="#1A73E8"
        android:layout_gravity="center"
        android:layout_marginBottom="20dp"/>

    <Button
        android:id="@+id/btnScan"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Run Privacy Scan"
        android:backgroundTint="#D93025"
        android:textColor="#FFFFFF"
        android:padding="12dp"/>

    <TextView
        android:id="@+id/statusText"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Ready to scan..."
        android:layout_marginTop="10dp"
        android:textColor="#5F6368"/>

    <ListView
        android:id="@+id/listView"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:layout_marginTop="10dp"/>
</LinearLayout>

// third code
package com.example.privasentry;

import android.os.Bundle;
import android.os.Environment;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.ListView;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;

import java.io.*;
import java.util.*;
import java.util.regex.*;

public class MainActivity extends AppCompatActivity {

    private ListView listView;
    private TextView statusText;
    private ArrayList<String> scanResults = new ArrayList<>();

    // Visa & MasterCard pattern
    private static final Pattern CARD_PATTERN =
            Pattern.compile("\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\\b");

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button btnScan = findViewById(R.id.btnScan);
        listView = findViewById(R.id.listView);
        statusText = findViewById(R.id.statusText);

        btnScan.setOnClickListener(v -> startScanInBackground());
    }

    // Run scan in background thread (no app freeze)
    private void startScanInBackground() {
        statusText.setText("Scanning Downloads folder...");

        new Thread(() -> {
            scanResults.clear();

            File downloads =
                    Environment.getExternalStoragePublicDirectory(
                            Environment.DIRECTORY_DOWNLOADS);

            scanDirectory(downloads);

            runOnUiThread(() -> {
                if (scanResults.isEmpty()) {
                    scanResults.add("✅ No sensitive data found. You're safe!");
                }

                listView.setAdapter(new ArrayAdapter<>(
                        this,
                        android.R.layout.simple_list_item_1,
                        scanResults));

                statusText.setText("Scan Completed");
            });

        }).start();
    }

    // Recursive folder scan
    private void scanDirectory(File dir) {
        if (dir == null || !dir.exists()) return;

        File[] files = dir.listFiles();
        if (files == null) return;

        for (File file : files) {
            if (file.isDirectory()) {
                scanDirectory(file);
            } else if (file.getName().endsWith(".txt") ||
                       file.getName().endsWith(".log")) {
                analyzeFile(file);
            }
        }
    }

    // Analyze file content
    private void analyzeFile(File file) {
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (CARD_PATTERN.matcher(line).find()) {
                    scanResults.add("⚠️ Warning: " + file.getName()
                            + " contains sensitive data!");
                    break;
                }
            }
        } catch (IOException ignored) {
        }
    }
}
