# monitoring-jaringan

#!/usr/bin/env python3
import subprocess
import time
from datetime import datetime
import re
import os

# Warna terminal
class Color:
    GREEN = "\033[92m"
    YELLOW = "\033[93m"
    RED = "\033[91m"
    CYAN = "\033[96m"
    RESET = "\033[0m"
    BOLD = "\033[1m"

def log_message(msg, log_file):
    """Cetak ke terminal dan simpan ke file log"""
    ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    entry = f"[{ts}] {msg}"
    print(entry)
    with open(log_file, "a") as f:
        f.write(entry + "\n")

def ping_once(host):
    """Ping 1x dan ambil waktu latency"""
    try:
        result = subprocess.run(
            ["ping", "-c", "1", "-W", "1", host],
            capture_output=True, text=True
        )
        if result.returncode == 0:
            match = re.search(r'time=(\d+\.\d+)\s*ms', result.stdout)
            if match:
                latency = float(match.group(1))
                return ("UP", latency)
        return ("DOWN", None)
    except Exception as e:
        return ("ERROR", str(e))

def evaluate_quality(latency):
    """Menilai kualitas koneksi berdasarkan latency"""
    if latency <= 50:
        return (f"{Color.GREEN}Sangat Bagus{Color.RESET}", "Koneksi cepat & stabil", "█" * 10)
    elif latency <= 100:
        return (f"{Color.CYAN}Bagus{Color.RESET}", "Koneksi stabil", "█" * 8 + "░" * 2)
    elif latency <= 200:
        return (f"{Color.YELLOW}Cukup{Color.RESET}", "Mulai terasa lambat", "█" * 6 + "░" * 4)
    elif latency <= 500:
        return (f"{Color.RED}Buruk{Color.RESET}", "Koneksi tidak stabil", "█" * 3 + "░" * 7)
    else:
        return (f"{Color.RED}Sangat Buruk{Color.RESET}", "Jaringan sangat lambat", "░" * 10)

def print_header(target, interval):
    os.system("clear")
    print(f"{Color.BOLD}{Color.CYAN}╔══════════════════════════════════════════════════════╗")
    print(f"║           PING QUALITY MONITOR PRO v3.0             ║")
    print(f"╚══════════════════════════════════════════════════════╝{Color.RESET}")
    print(f"Target   : {target}")
    print(f"Interval : {interval} detik")
    print(f"Mulai    : {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'-'*55}\n")

def main():
    # ======== Input Target dari Pengguna =========
    print(f"{Color.BOLD}{Color.CYAN}=== PING QUALITY MONITOR PRO v3.0 ==={Color.RESET}")
    target = input(f"Masukkan target (contoh: www.netacad.com atau 8.8.8.8)\nTekan ENTER untuk default [www.google.com]: ").strip()
    if not target:
        target = "www.google.com"

    interval_input = input("Masukkan interval pengecekan (detik, default 5): ").strip()
    interval = int(interval_input) if interval_input.isdigit() else 5

    log_file = f"/var/log/ping_monitor_{target.replace('.', '_')}.log"

    success_count = 0
    fail_count = 0
    latency_sum = 0.0
    total_ping = 0

    print_header(target, interval)
    log_message(f"=== Memulai monitoring ke {target} ===", log_file)

    while True:
        total_ping += 1
        status, result = ping_once(target)

        if status == "UP":
            latency = result
            success_count += 1
            latency_sum += latency
            avg_latency = latency_sum / success_count

            quality, desc, bar = evaluate_quality(latency)

            os.system("clear")
            print_header(target, interval)
            print(f"{Color.BOLD}[STATUS] {Color.GREEN}ONLINE{Color.RESET}")
            print(f"[WAKTU]   {datetime.now().strftime('%H:%M:%S')}")
            print(f"[PING]    {latency:.2f} ms   (Rata-rata: {avg_latency:.2f} ms)")
            print(f"[KUALITAS] {quality} → {desc}")
            print(f"[GRAFIK]   {bar}")
            print(f"[PAKET]   Total: {total_ping} | Sukses: {success_count} | Gagal: {fail_count} | Loss: {(fail_count/total_ping)*100:.1f}%")
            print(f"{'-'*55}")

            log_message(f"[OK] {target} → {latency:.2f} ms | {quality} | {desc}", log_file)

        elif status == "DOWN":
            fail_count += 1
            os.system("clear")
            print_header(target, interval)
            print(f"{Color.BOLD}[STATUS] {Color.RED}OFFLINE / TIMEOUT{Color.RESET}")
            print(f"[WAKTU]   {datetime.now().strftime('%H:%M:%S')}")
            print(f"[PING]    Gagal")
            print(f"[KUALITAS] {Color.RED}Terputus{Color.RESET} → Tidak ada respon")
            print(f"[GRAFIK]   {'░'*10}")
            print(f"[PAKET]   Total: {total_ping} | Sukses: {success_count} | Gagal: {fail_count} | Loss: {(fail_count/total_ping)*100:.1f}%")
            print(f"{'-'*55}")

            log_message(f"[ERROR] {target} tidak terjangkau!", log_file)

        else:
            log_message(f"[ERROR] Exception: {result}", log_file)

        time.sleep(interval)

if __name__ == "__main__":
    main()
