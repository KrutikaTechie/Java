<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.privasentry">

    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="PrivaSentry"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.Light.DarkActionBar">
        
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>

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
        android:layout_gravity="center"
        android:textColor="#1A73E8"
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
        android:layout_marginTop="10dp"
        android:divider="#E0E0E0"
        android:dividerHeight="1dp"/>
</LinearLayout>

//java code 
package com.example.privasentry;

import android.content.Intent;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Environment;
import android.provider.Settings;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.ListView;
import android.widget.TextView;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

import java.io.*;
import java.util.*;
import java.util.regex.*;

public class MainActivity extends AppCompatActivity {
    private ListView listView;
    private TextView statusText;
    private ArrayList<String> scanResults = new ArrayList<>();
    
    // Regex to find potential Credit Card Numbers
    private static final Pattern CARD_PATTERN = Pattern.compile("\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\\b");

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button btnScan = findViewById(R.id.btnScan);
        listView = findViewById(R.id.listView);
        statusText = findViewById(R.id.statusText);

        btnScan.setOnClickListener(v -> {
            if (checkStoragePermission()) {
                startPrivacyScan();
            } else {
                requestStoragePermission();
            }
        });
    }

    private void startPrivacyScan() {
        scanResults.clear();
        statusText.setText("Scanning Downloads folder...");
        
        // Target the Downloads directory
        File downloadsDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS);
        
        scanDirectory(downloadsDir);
        
        if(scanResults.isEmpty()){
            scanResults.add("No threats found. You're safe!");
        }

        ArrayAdapter<String> adapter = new ArrayAdapter<>(this, android.R.layout.simple_list_item_1, scanResults);
        listView.setAdapter(adapter);
        statusText.setText("Scan Complete!");
    }

    private void scanDirectory(File directory) {
        File[] files = directory.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isDirectory()) {
                    scanDirectory(file); // Recursive call for sub-folders
                } else if (file.getName().endsWith(".txt") || file.getName().endsWith(".log")) {
                    analyzeFile(file);
                }
            }
        }
    }

    private void analyzeFile(File file) {
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (CARD_PATTERN.matcher(line).find()) {
                    scanResults.add("⚠️ WARNING: " + file.getName() + " contains Credit Card info!");
                    break; 
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Permission handling for Android 11+
    private boolean checkStoragePermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            return Environment.isExternalStorageManager();
        }
        return true; // Simple check for older versions
    }

    private void requestStoragePermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            Intent intent = new Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION);
            intent.setData(Uri.parse("package:" + getPackageName()));
            startActivity(intent);
        } else {
            Toast.makeText(this, "Please enable storage permissions in Settings", Toast.LENGTH_LONG).show();
        }
    }
}
