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

public class NativeExecutor {

    // ---------------------------------------------------------
    // [LABEL: VARIABEL GLOBAL & UI COMPONENTS]
    // ---------------------------------------------------------
    private Activity activity;
    private Handler uiHandler;

    private TextView terminalOutput;
    private ScrollView terminalScroll;
    private EditText terminalInput;

    private File filesDir;
    private File usrDir;

    private Process shellProcess;
    private BufferedWriter shellInput;

    // ---------------------------------------------------------
    // [LABEL: CONSTRUCTOR - INISIALISASI AWAL]
    // ---------------------------------------------------------
    public NativeExecutor(Activity activity) {
        this.activity = activity;
        this.uiHandler = new Handler(Looper.getMainLooper());

        // Menghubungkan variabel dengan ID XML secara dinamis
        terminalOutput = (TextView) activity.findViewById(
                activity.getResources().getIdentifier("terminal_output", "id", activity.getPackageName())
        );

        terminalScroll = (ScrollView) activity.findViewById(
                activity.getResources().getIdentifier("terminal_scroll", "id", activity.getPackageName())
        );

        terminalInput = (EditText) activity.findViewById(
                activity.getResources().getIdentifier("terminal_input", "id", activity.getPackageName())
        );

        // Inisialisasi lokasi direktori internal aplikasi
        filesDir = activity.getFilesDir();
        usrDir = new File(filesDir, "usr");

        setupInput();
    }

    // ---------------------------------------------------------
    // [LABEL: METHOD START - TITIK AWAL EKSEKUSI]
    // ---------------------------------------------------------
    public void start() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    // Ekstraksi binary pendukung jika belum ada
                    if (!usrDir.exists()) {
                        appendOutput("[*] Extract usr.zip...");
                        unzipFromAssets("usr.zip", filesDir);
                    }

                    // Memberikan izin eksekusi (chmod 755) pada folder bin
                    fixExecutable(usrDir);
                    
                    // Melanjutkan ke instalasi/setup distro Alpine
                    setupAlpineProotDistro();

                } catch (Exception e) {
                    appendOutput("[ERROR] " + e.getMessage());
                }
            }
        }).start();
    }

    // ---------------------------------------------------------
    // [LABEL: START PROOT SHELL - MENJALANKAN LINUX ENVIRONMENT]
    // ---------------------------------------------------------
    private void startProotShell() throws Exception {

        File proot = new File(usrDir, "bin/proot");
        File libDir = new File(usrDir, "lib");
        File tmpDir = new File(usrDir, "tmp");
        File rootfs = new File(usrDir, "var/lib/proot-distro/installed-rootfs/alpine");

        tmpDir.mkdirs();

        // Konfigurasi mounting dan root directory untuk PRoot
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
        
        // Pengaturan Environment Variables (PATH, HOME, LD_LIBRARY_PATH)
        Map<String, String> env = pb.environment();
        env.put("LD_LIBRARY_PATH", libDir.getAbsolutePath());
        env.put("HOME", "/root");
        env.put("USER", "root");
        env.put("TMPDIR", "/tmp");
        env.put("PWD", "/root");

        // kalau pake ubuntu/debian
        //env.put("DEBIAN_FRONTEND", "noninteractive");
        env.put("PATH", "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin");
        env.put("PROOT_TMP_DIR", tmpDir.getAbsolutePath());
        env.put("PROOT_LOADER", usrDir.getAbsolutePath() + "/libexec/proot/loader");
        env.put("PROOT_LOADER_32", usrDir.getAbsolutePath() + "/libexec/proot/loader32");
        env.put("TERM", "xterm-256color");

        // Menjalankan proses shell
        shellProcess = pb.start();
        shellInput = new BufferedWriter(new OutputStreamWriter(shellProcess.getOutputStream()));

        // Membaca output (Standard Output & Error) secara real-time
        readStream(shellProcess.getInputStream());
        readStream(shellProcess.getErrorStream());

        appendOutput("[✓] PRoot shell ready!");
    }

    // ---------------------------------------------------------
    // [LABEL: SETUP INPUT - MENGATUR LISTENER KEYBOARD]
    // ---------------------------------------------------------
    private void setupInput() {
        terminalInput.setOnEditorActionListener(new TextView.OnEditorActionListener() {
            @Override
            public boolean onEditorAction(TextView v, int actionId, KeyEvent event) {
                // Menangkap aksi "Send" atau tombol "Enter"
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
            }
        });
    }

    // ---------------------------------------------------------
    // [LABEL: SETUP ALPINE - INSTALASI DISTRO LINUX]
    // ---------------------------------------------------------
    private void setupAlpineProotDistro() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    File proot = new File(usrDir, "bin/proot");
                    File libDir = new File(usrDir, "lib");
                    File tmpDir = new File(usrDir, "tmp");
                    
                    File baseDir = new File(usrDir, "var/lib/proot-distro/installed-rootfs/alpine");
                    File tarGz = new File(filesDir, "alpine.tar.gz");
                    File tarBinary = new File(usrDir, "bin/tar");
                    

                    // 1. Cek apakah sudah terinstall
                    if (baseDir.exists() && baseDir.list() != null && baseDir.list().length > 0) {
                        appendOutput("[*] Alpine already installed.");
                        appendOutput("[*] Starting PRoot shell...");
                        startProotShell();
                        return;
                    }

                    // 2. Buat direktori tujuan
                    appendOutput("[*] Creating rootfs directory...");
                    if (!baseDir.exists()) baseDir.mkdirs();

                    // 3. Download file rootfs (jika belum ada di storage)
                    if (!tarGz.exists()) {
                        String url = "https://dl-cdn.alpinelinux.org/alpine/v3.23/releases/aarch64/alpine-minirootfs-3.23.0-aarch64.tar.gz";
                        appendOutput("[*] Downloading Alpine rootfs...");
                        downloadFile(url, tarGz);
                    }

                    // 4. Ekstrak menggunakan binary TAR native
                    appendOutput("[*] Extracting Alpine using native tar...");
                    
                    ProcessBuilder pb = new ProcessBuilder(
    proot.getAbsolutePath(),
    "--link2symlink",   // 🔥 ini kunci!
    tarBinary.getAbsolutePath(),
    "-xzf",
    tarGz.getAbsolutePath(),
    "-C",
    baseDir.getAbsolutePath()
);

                    // Menyuntikkan library path agar tar bisa berjalan
                    Map<String, String> env = pb.environment();
                    env.put("LD_LIBRARY_PATH", libDir.getAbsolutePath());
                    env.put("PROOT_TMP_DIR", tmpDir.getAbsolutePath());
                    env.put("PROOT_LOADER", usrDir.getAbsolutePath() + "/libexec/proot/loader");
                    env.put("PROOT_LOADER_32", usrDir.getAbsolutePath() + "/libexec/proot/loader32");

                    Process p = pb.start();
                    readStream(p.getInputStream());
                    readStream(p.getErrorStream());

                    int exitCode = p.waitFor();

                    if (exitCode == 0) {
                        appendOutput("[✓] Alpine extraction successful!");
                        appendOutput("[✓] Setup complete at: " + baseDir.getAbsolutePath());
                        
                        // Konfigurasi DNS (resolv.conf) agar internet dalam shell lancar
                        File resolv = new File(baseDir, "etc/resolv.conf");
                        FileWriter fw = new FileWriter(resolv);
                        fw.write("nameserver 8.8.8.8\n");
                        fw.write("nameserver 1.1.1.1\n");
                        fw.close();

                        // Jalankan shell setelah install selesai
                        startProotShell();
                    } else {
                        appendOutput("[ERROR] Tar exited with code: " + exitCode);
                    }

                } catch (Exception e) {
                    appendOutput("[ERROR] Setup failed: " + e.getMessage());
                    e.printStackTrace();
                }
            }
        }).start();
    }

    // ---------------------------------------------------------
    // [LABEL: DOWNLOAD FILE - FUNGSI PENGUNDUH]
    // ---------------------------------------------------------
    private void downloadFile(String urlStr, File output) throws Exception {
        java.net.URL url = new java.net.URL(urlStr);
        java.net.HttpURLConnection conn = (java.net.HttpURLConnection) url.openConnection();
        conn.connect();

        InputStream in = conn.getInputStream();
        FileOutputStream fos = new FileOutputStream(output);

        byte[] buffer = new byte[8192];
        int len;
        while ((len = in.read(buffer)) != -1) {
            fos.write(buffer, 0, len);
        }

        fos.close();
        in.close();
    }

    // ---------------------------------------------------------
    // [LABEL: SEND COMMAND - MENGIRIM PERINTAH KE SHELL]
    // ---------------------------------------------------------
    private void sendCommand(String cmd) {
        try {
            appendOutput("$ " + cmd);
            if (shellInput != null) {
                shellInput.write(cmd);
                shellInput.newLine();
                shellInput.flush();
            }
        } catch (Exception e) {
            appendOutput("[ERROR] " + e.getMessage());
        }
    }

    // ---------------------------------------------------------
    // [LABEL: FIX EXECUTABLE - MENGATUR PERMISSION BINARY]
    // ---------------------------------------------------------
    private void fixExecutable(File dir) {
        try {
            appendOutput("[*] Fixing permissions via chmod -R 755...");
            String[] cmd = {"/system/bin/chmod", "-R", "755", dir.getAbsolutePath()};
            Runtime.getRuntime().exec(cmd).waitFor();
        } catch (Exception e) {
            appendOutput("[ERROR] chmod failed: " + e.getMessage());
        }
    }

    // ---------------------------------------------------------
    // [LABEL: UNZIP ASSET - EKSTRAKSI ZIP DARI FOLDER ASSETS]
    // ---------------------------------------------------------
    private void unzipFromAssets(String zipName, File target) throws IOException {
        ZipInputStream zis = new ZipInputStream(activity.getAssets().open(zipName));
        ZipEntry ze;

        while ((ze = zis.getNextEntry()) != null) {
            File f = new File(target, ze.getName());

            if (ze.isDirectory()) {
                f.mkdirs();
            } else {
                f.getParentFile().mkdirs();
                FileOutputStream fos = new FileOutputStream(f);
                byte[] buf = new byte[8192];
                int len;
                while ((len = zis.read(buf)) > 0) {
                    fos.write(buf, 0, len);
                }
                fos.close();
            }
        }
        zis.close();
    }

    // ---------------------------------------------------------
    // [LABEL: READ STREAM - MEMBACA OUTPUT PROSES]
    // ---------------------------------------------------------
    private void readStream(final InputStream is) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    BufferedReader br = new BufferedReader(new InputStreamReader(is));
                    String line;
                    while ((line = br.readLine()) != null) {
                        appendOutput(line);
                    }
                    br.close();
                } catch (Exception ignored) {}
            }
        }).start();
    }

    // ---------------------------------------------------------
    // [LABEL: APPEND OUTPUT - MENAMPILKAN TEKS KE UI]
    // ---------------------------------------------------------
    private void appendOutput(final String text) {
        uiHandler.post(new Runnable() {
            @Override
            public void run() {
                terminalOutput.append(text + "\n");
                // Auto-scroll ke bawah setiap kali ada teks baru
                terminalScroll.post(new Runnable() {
                    @Override
                    public void run() {
                        terminalScroll.fullScroll(View.FOCUS_DOWN);
                    }
                });
            }
        });
    }
}
