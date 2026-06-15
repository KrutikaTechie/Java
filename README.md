package com.example.privasentry;

import android.Manifest;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.os.Environment;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.ListView;
import android.widget.TextView;
import android.widget.Toast;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import java.io.*;
import java.util.*;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.regex.*;

public class MainActivity extends AppCompatActivity {
    private static final int PERMISSION_REQUEST_CODE = 101;
    private ListView listView;
    private TextView statusText;
    private ArrayList<String> scanResults = new ArrayList<>();
    private ArrayAdapter<String> adapter;
    private final ExecutorService executorService = Executors.newSingleThreadExecutor();
// Enhanced Detection Patterns
    private static final Pattern CARD_PATTERN =
            Pattern.compile("\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\\b");
    private static final Pattern EMAIL_PATTERN = 
            Pattern.compile("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,6}");
    private static final Pattern PASSWORD_KEY_PATTERN = 
            Pattern.compile("(?i)(password|passwd|api_key|secret)\\s*[:=]\\s*['\"][a-zA-Z0-9@#$%^&*()_+]{4,15}['\"]");
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button btnScan = findViewById(R.id.btnScan);
        listView = findViewById(R.id.listView);
        statusText = findViewById(R.id.statusText);
        // Initialize adapter cleanly
        adapter = new ArrayAdapter<>(this, android.R.layout.simple_list_item_1, scanResults);
        listView.setAdapter(adapter);
        btnScan.setOnClickListener(v -> checkPermissionAndScan());
    }
// Step 1: Check if the app has storage access permission at runtime
    private void checkPermissionAndScan() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_EXTERNAL_STORAGE)
                == PackageManager.PERMISSION_GRANTED) {
            startScanInBackground();
        } else {
            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.READ_EXTERNAL_STORAGE}, PERMISSION_REQUEST_CODE);
        }
    }
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == PERMISSION_REQUEST_CODE) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                startScanInBackground();
            } else {
                Toast.makeText(this, "Permission Denied! Cannot scan storage.", Toast.LENGTH_LONG).show();
                statusText.setText("Permission required to scan files.");
            }
        }
    }
    // Step 2: Run scan via ExecutorService instead of loose unmanaged threads
    private void startScanInBackground() {
        statusText.setText("Scanning Downloads folder...");
        scanResults.clear();
        adapter.notifyDataSetChanged();
        executorService.execute(() -> {
            File downloads = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS);
            scanDirectory(downloads);
            runOnUiThread(() -> {
                if (scanResults.isEmpty()) {
                    scanResults.add("✅ No sensitive data found. Your logs are safe!");
                }
                adapter.notifyDataSetChanged();
                statusText.setText("Scan Completed");
            });
        });
    }
    private void scanDirectory(File dir) {
        if (dir == null || !dir.exists()) return;
        File[] files = dir.listFiles();
        if (files == null) return;
        for (File file : files) {
            if (file.isDirectory()) {
                scanDirectory(file);
            } else if (file.getName().endsWith(".txt") || file.getName().endsWith(".log")) {
                analyzeFile(file);
            }
        }
    }
    // Step 3: Analyze content against multiple security risk vectors
    private void analyzeFile(File file) {
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            String line;
            int lineNumber = 0;
            while ((line = reader.readLine()) != null) {
                lineNumber++;
                String finding = null;
                if (CARD_PATTERN.matcher(line).find()) {
                    finding = "💳 Credit Card Info";
                } else if (PASSWORD_KEY_PATTERN.matcher(line).find()) {
                    finding = "🔑 Plaintext Password/API Key";
                } else if (EMAIL_PATTERN.matcher(line).find()) {
                    finding = "📧 PII Exposure (Email)";
                }
                if (finding != null) {
                    scanResults.add("⚠️ " + finding + " found in " + file.getName() + " (Line " + lineNumber + ")");
                    break; // Found an issue in this file, jump to next file
                }
            }
        } catch (IOException ignored) {
        }
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        executorService.shutdown(); // Prevent memory leaks when app closes
    }
}
