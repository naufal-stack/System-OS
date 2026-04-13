# 🚀 Android Native Linux Executor (PRoot-based)

[![Android](https://img.shields.io/badge/Platform-Android-brightgreen?logo=android)](https://developer.android.com)
[![Linux](https://img.shields.io/badge/Distro-Alpine%20Linux-blueviolet?logo=alpine-linux)](https://alpinelinux.org/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Sebuah modul Java canggih untuk menjalankan **Alpine Linux Environment** di dalam aplikasi Android tanpa akses Root. Library ini menggunakan **PRoot** untuk memetakan file system dan menjalankan shell Linux langsung di dalam UI Android kamu.

---

## ✨ Fitur Utama

* **Zero Root**: Berjalan sepenuhnya di user-space.
* **Automated Setup**: Mengunduh, mengekstrak, dan mengonfigurasi *rootfs* secara otomatis.
* **Real-time Terminal**: Integrasi langsung antara output proses Linux dan UI Android (`TextView`).
* **Network Ready**: Konfigurasi DNS otomatis untuk akses internet di dalam shell.
* **Native Performance**: Menggunakan binary `tar` dan `proot` asli untuk kecepatan maksimal.

---

## 📂 Struktur File Pendukung

Agar modul ini bekerja, kamu perlu meletakkan beberapa file di folder `assets/`:
1.  **`usr.zip`**: Berisi folder `bin`, `lib`, dan `libexec` dari PRoot.
2.  **`alpine.tar.gz`**: (Opsional) Jika tidak ingin mengunduh saat pertama kali dijalankan.

---

## 💻 Implementasi Kode

Gunakan kelas `NativeExecutor.java` berikut di dalam project Android kamu:

### NativeExecutor.java
```java
package com.my.newproject3;

import android.app.Activity;
import android.os.Handler;
import android.os.Looper;
import android.view.KeyEvent;
import android.view.View;
import android.view.inputmethod.EditorInfo;
import android.widget.*;

import java.io.*;
import java.util.Map;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

/**
 * NativeExecutor - Menjalankan Linux Distro via PRoot di Android.
 * Didesain untuk integrasi terminal yang estetik.
 */
public class NativeExecutor {

    private Activity activity;
    private Handler uiHandler;
    private TextView terminalOutput;
    private ScrollView terminalScroll;
    private EditText terminalInput;
    private File filesDir;
    private File usrDir;
    private Process shellProcess;
    private BufferedWriter shellInput;

    public NativeExecutor(Activity activity) {
        this.activity = activity;
        this.uiHandler = new Handler(Looper.getMainLooper());

        // Inisialisasi UI secara dinamis berdasarkan Resource ID
        terminalOutput = (TextView) activity.findViewById(
                activity.getResources().getIdentifier("terminal_output", "id", activity.getPackageName())
        );
        terminalScroll = (ScrollView) activity.findViewById(
                activity.getResources().getIdentifier("terminal_scroll", "id", activity.getPackageName())
        );
        terminalInput = (EditText) activity.findViewById(
                activity.getResources().getIdentifier("terminal_input", "id", activity.getPackageName())
        );

        filesDir = activity.getFilesDir();
        usrDir = new File(filesDir, "usr");

        setupInput();
    }

    public void start() {
        new Thread(() -> {
            try {
                if (!usrDir.exists()) {
                    appendOutput("[*] Memulai ekstraksi usr.zip...");
                    unzipFromAssets("usr.zip", filesDir);
                }
                fixExecutable(usrDir);
                setupAlpineProotDistro();
            } catch (Exception e) {
                appendOutput("[ERROR] Inisialisasi gagal: " + e.getMessage());
            }
        }).start();
    }

    private void startProotShell() throws Exception {
        File proot = new File(usrDir, "bin/proot");
        File libDir = new File(usrDir, "lib");
        File tmpDir = new File(usrDir, "tmp");
        File rootfs = new File(usrDir, "var/lib/proot-distro/installed-rootfs/alpine");

        tmpDir.mkdirs();

        ProcessBuilder pb = new ProcessBuilder(
            proot.getAbsolutePath(),
            "--link2symlink",
            "--kill-on-exit",
            "-0",
            "-r", rootfs.getAbsolutePath(),
            "-b", "/dev",
            "-b", "/proc",
            "-b", "/sys",
            "-b", "/system",
            "-b", "/vendor",
            "-b", usrDir.getAbsolutePath() + "/tmp:/tmp",
            "-w", "/root",
            "/bin/sh"
        );
        
        Map<String, String> env = pb.environment();
        env.put("LD_LIBRARY_PATH", libDir.getAbsolutePath());
        env.put("HOME", "/root");
        env.put("PATH", "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin");
        env.put("PROOT_TMP_DIR", tmpDir.getAbsolutePath());
        env.put("PROOT_LOADER", usrDir.getAbsolutePath() + "/libexec/proot/loader");
        env.put("TERM", "xterm-256color");

        shellProcess = pb.start();
        shellInput = new BufferedWriter(new OutputStreamWriter(shellProcess.getOutputStream()));

        readStream(shellProcess.getInputStream());
        readStream(shellProcess.getErrorStream());

        appendOutput("[✓] Shell Linux Siap!");
    }

    private void setupInput() {
        terminalInput.setOnEditorActionListener((v, actionId, event) -> {
            if (actionId == EditorInfo.IME_ACTION_SEND ||
                    (event != null && event.getKeyCode() == KeyEvent.KEYCODE_ENTER)) {
                String cmd = terminalInput.getText().toString().trim();
                if (!cmd.isEmpty()) {
                    sendCommand(cmd);
                    terminalInput.setText("");
                }
                return true;
            }
            return false;
        });
    }

    private void sendCommand(String cmd) {
        try {
            appendOutput("$ " + cmd);
            if (shellInput != null) {
                shellInput.write(cmd);
                shellInput.newLine();
                shellInput.flush();
            }
        } catch (Exception e) {
            appendOutput("[ERROR] Gagal mengirim perintah: " + e.getMessage());
        }
    }

    private void appendOutput(final String text) {
        uiHandler.post(() -> {
            terminalOutput.append(text + "\n");
            terminalScroll.post(() -> terminalScroll.fullScroll(View.FOCUS_DOWN));
        });
    }

    // ... (Metode unzipFromAssets, fixExecutable, readStream, setupAlpine tetap sama)
}
