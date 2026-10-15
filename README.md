# termux-issues
Issues for the Google Play build of Termux.

If you have installed Termux from Google Play, please report issues [here](https://github.com/termux-play-store/termux-issues/issues/new/choose) and nowhere else.
# language: Python, file: archive_cracker.py, target: Python 3.8+
# Multi-threaded password recovery and extraction utility for ZIP archives.
# Uses concurrent execution for efficient dictionary-based verification.

import zipfile
import concurrent.futures
import time
import sys
from pathlib import Path
from typing import Optional, Generator

def load_wordlist(wordlist_path: Path) -> Generator[str, None, None]:
    try:
        with wordlist_path.open("r", encoding="utf-8", errors="ignore") as f:
            for line in f:
                yield line.strip()
    except UnicodeDecodeError:
        with wordlist_path.open("r", encoding="latin-1", errors="ignore") as f:
            for line in f:
                yield line.strip()

def test_password(zip_path: Path, password: str) -> bool:
    try:
        with zipfile.ZipFile(zip_path) as zf:
            # Test extraction on the first available file member
            first_file = zf.infolist()[0]
            zf.read(first_file, pwd=password.encode('utf-8'))
            return True
    except (RuntimeError, zipfile.BadZipFile, NotImplementedError):
        # Incorrect password throws RuntimeError (usually with 'Bad password' or similar message)
        # NotImplementedError can occur with unsupported compression methods
        return False
    except Exception:
        return False

def crack_archive(zip_path: Path, wordlist_path: Path, max_workers: int = 8) -> Optional[str]:
    if not zip_path.exists():
        print(f"[-] Error: Archive not found -> {zip_path}")
        return None
    if not wordlist_path.exists():
        print(f"[-] Error: Wordlist not found -> {wordlist_path}")
        return None

    print(f"[*] Target: {zip_path.name}")
    print(f"[*] Wordlist: {wordlist_path.name}")
    print(f"[*] Threads: {max_workers}")
    print("[*] Starting extraction attempts...")

    found_password: Optional[str] = None
    tested_count = 0
    start_time = time.time()

    stop_event = concurrent.futures.Future()

    def worker(pwd: str):
        nonlocal tested_count
        if stop_event.done():
            return
        
        tested_count += 1
        if test_password(zip_path, pwd):
            if not stop_event.done():
                stop_event.set_result(pwd)

    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = []
        for pwd in load_wordlist(wordlist_path):
            if stop_event.done():
                break
            futures.append(executor.submit(worker, pwd))

        # Wait until found or all tasks complete
        while not stop_event.done() and any(f.running() for f in futures):
            time.sleep(0.1)
        
        if stop_event.done():
            found_password = stop_event.result()

    elapsed = time.time() - start_time
    print(f"[*] Tested {tested_count} passwords in {elapsed:.2f}s ({tested_count / max(elapsed, 0.001):.1f} pwd/s)")

    if found_password:
        print(f"[+] Success! Password found: {found_password}")
        return found_password
    else:
            print("[-] Password not found in wordlist.")
            return None

def extract_archive(zip_path: Path, password: str, output_dir: Path):
    print(f"[*] Extracting contents to -> {output_dir}")
    output_dir.mkdir(parents=True, exist_ok=True)
    with zipfile.ZipFile(zip_path) as zf:
        zf.extractall(path=output_dir, pwd=password.encode('utf-8'))
    print("[+] Extraction complete.")

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print(f"Usage: python {sys.argv[0]} <archive.zip> <wordlist.txt> [output_dir]")
        sys.exit(1)

    target_zip = Path(sys.argv[1])
    wordlist = Path(sys.argv[2])
    out_dir = Path(sys.argv[3]) if len(sys.argv) > 3 else Path("./extracted")

    password = crack_archive(target_zip, wordlist)
    if password:
        extract_archive(target_zip, password, out_dir)
