import subprocess
import os

def check_root():
    """Ensures the script is run with sudo privileges."""
    if os.geteuid() != 0:
        print("[ERROR] This script must be run with root privileges! Use: sudo python3 wifi_controller.py")
        exit(1)

def get_connected_devices():
    """Scans the local ARP table to list connected devices (IP and MAC)."""
    print("\n--- Scanning Connected Devices ---")
    devices = []
    try:
        # Read the Linux ARP table
        with open("/proc/net/arp", "r") as f:
            lines = f.readlines()[1:] # Skip header
            for line in lines:
                parts = line.split()
                if len(parts) >= 6:
                    ip = parts[0]
                    mac = parts[3]
                    if mac != "00:00:00:00:00:00":
                        devices.append({"ip": ip, "mac": mac})
        
        if not devices:
            print("No active devices found.")
        else:
            print(f"{'No.':<4} {'IP Address':<15} {'MAC Address':<18}")
            print("-" * 40)
            for idx, dev in enumerate(devices, 1):
                print(f"{idx:<4} {dev['ip']:<15} {dev['mac']:<18}")
        return devices
    except Exception as e:
        print(f"Error reading devices: {e}")
        return []

def block_device(mac_address):
    """Blocks a device's internet access by its MAC address."""
    try:
        cmd = ["iptables", "-A", "FORWARD", "-m", "mac", "--mac-source", mac_address, "-j", "DROP"]
        subprocess.run(cmd, check=True)
        print(f"\n[SUCCESS] Device {mac_address} has been BLOCKED from the internet.")
    except subprocess.CalledProcessError as e:
        print(f"\n[ERROR] Could not block device: {e}")

def unblock_device(mac_address):
    """Restores internet access for a blocked MAC address."""
    try:
        cmd = ["iptables", "-D", "FORWARD", "-m", "mac", "--mac-source", mac_address, "-j", "DROP"]
        subprocess.run(cmd, check=True)
        print(f"\n[SUCCESS] Device {mac_address} has been UNBLOCKED.")
    except subprocess.CalledProcessError as e:
        print(f"\n[ERROR] Could not unblock device (it might not be currently blocked): {e}")

def main():
    check_root()
    
    while True:
        print("\n==============================")
        print("   WI-FI CONTROL MANAGER      ")
        print("==============================")
        print("1. Scan Connected Devices")
        print("2. Block a Device (by MAC)")
        print("3. Unblock a Device (by MAC)")
        print("4. Exit")
        
        choice = input("\nSelect an option (1-4): ").strip()
        
        if choice == "1":
            get_connected_devices()
        elif choice == "2":
            mac = input("Enter the MAC address to block (e.g., AA:BB:CC:DD:EE:FF): ").strip()
            if mac:
                block_device(mac)
        elif choice == "3":
            mac = input("Enter the MAC address to unblock: ").strip()
            if mac:
                unblock_device(mac)
        elif choice == "4":
            print("Exiting Wi-Fi Controller. Goodbye!")
            break
        else:
            print("[INVALID] Please choose a valid option between 1 and 4.")

if __name__ == "__main__":
    main()

