# python
import tkinter as tk
from tkinter import messagebox, simpledialog
import json
from datetime import datetime

# 1. DATA INITIALIZATION
# Try to load existing user data from a JSON file
try:
    with open("bank_data.json", "r") as file:
        users = json.load(file)
except FileNotFoundError:
    users = {}  # Create empty database if file doesn't exist

# Ensure all existing users have the correct data structure (Backwards Compatibility)
for uname, info in users.items():
    if "balance" not in info:
        info["balance"] = 0.0
    if "loan" not in info:
        info["loan"] = {"unpaid": 0.0, "monthly": 0.0}


# 2. HELPER FUNCTIONS
def save_data():
    """Saves the current 'users' dictionary to the JSON file."""
    with open("bank_data.json", "w") as file:
        json.dump(users, file, indent=4)


def log_transaction(username, action, amount=0, extra=""):
    """Appends a timestamped record to the master transaction text file."""
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open("transaction_log.txt", "a", encoding="utf-8") as log:
        log.write(f"[{now}] {username} - {action} - ₱{amount:.2f} {extra}\n")


def export_user_log(username):
    """Filters the master log for one user and saves it to a personal file."""
    try:
        with open("transaction_log.txt", "r", encoding="utf-8") as file:
            lines = file.readlines()

        found = False
        with open(f"{username}_transactions.txt", "w", encoding="utf-8") as out:
            for line in lines:
                if f" {username} " in line:
                    out.write(line)
                    found = True

        if found:
            messagebox.showinfo("Export", f"History saved to {username}_transactions.txt")
        else:
            messagebox.showinfo("Export", "No history found.")
    except FileNotFoundError:
        messagebox.showerror("Error", "No log file found.")


# 3. ADMIN INTERFACE
def admin_panel():
    """Special menu for administrators to monitor the entire system."""
    while True:
        choice = simpledialog.askstring("ADMIN", "1-View Users\n2-Balances\n3-View Logs\n4-Export All\n5-Exit")
        if choice == "5" or choice is None: break

        if choice == "1":
            messagebox.showinfo("Users", "\n".join(users.keys()) if users else "No users.")
        elif choice == "2":
            text = "\n".join([f"{u}: ₱{info['balance']:.2f}" for u, info in users.items()])
            messagebox.showinfo("Balances", text if text else "No data.")
        elif choice == "3":
            try:
                with open("transaction_log.txt", "r") as f:
                    messagebox.showinfo("Logs", f.read())
            except:
                messagebox.showerror("Error", "No logs.")
        elif choice == "4":
            save_data()  # Ensure latest data is saved
            messagebox.showinfo("Export", "System data synchronized.")


# 4. LOGIN & AUTHENTICATION
root = tk.Tk()
root.withdraw()  # Hide the main empty Tkinter window

username = simpledialog.askstring("Bank", "Enter Name:")
pin = simpledialog.askstring("Bank", "Enter PIN:")

if not username or not pin: exit()

# Check for Admin Credentials
if username == "admin" and pin == "admin123":
    admin_panel()
    exit()

# Handle User Login or Registration
if username in users:
    if users[username]["pin"] != pin:
        messagebox.showerror("Denied", "Wrong PIN.")
        exit()
else:
    # Register New User
    users[username] = {"pin": pin, "balance": 0.0, "loan": {"unpaid": 0.0, "monthly": 0.0}}
    save_data()
    messagebox.showinfo("Welcome", "Account Created!")

# Local reference for easier data manipulation
user_ref = users[username]

# 5. MAIN TRANSACTION LOOP
while True:
    choice = simpledialog.askstring(
        "Bank Menu",
        f"User: {username}\n1-Balance\n2-Deposit\n3-Withdraw\n4-Loan\n5-Pay Loan\n6-Transfer\n7-Export\n8-Exit"
    )

    if choice == "8" or choice is None:
        save_data()
        break

    # 1. CHECK BALANCE
    if choice == "1":
        msg = f"Balance: ₱{user_ref['balance']:.2f}"
        if user_ref['loan']['unpaid'] > 0:
            msg += f"\nLoan Debt: ₱{user_ref['loan']['unpaid']:.2f}"
        messagebox.showinfo("Account Info", msg)

    # 2. DEPOSIT
    elif choice == "2":
        amt = simpledialog.askfloat("Deposit", "Amount:")
        if amt and amt > 0:
            user_ref["balance"] += amt
            log_transaction(username, "DEPOSIT", amt)
            save_data()

    # 3. WITHDRAW
    elif choice == "3":
        amt = simpledialog.askfloat("Withdraw", "Amount:")
        if amt and 0 < amt <= user_ref["balance"]:
            user_ref["balance"] -= amt
            log_transaction(username, "WITHDRAW", amt)
            save_data()
        else:
            messagebox.showwarning("Error", "Invalid amount or insufficient funds.")

    # 4. APPLY FOR LOAN
    elif choice == "4":
        if user_ref["loan"]["unpaid"] > 0:
            messagebox.showwarning("Loan", "Pay your existing loan first!")
            continue

        amt = simpledialog.askfloat("Loan", "Enter Amount (10k, 20k, 30k, 40k, 50k, or 60k-100k):")
        # Logic for interest rates based on amount
        rates = {10000: 10, 20000: 20, 30000: 30, 40000: 40, 50000: 50}
        rate = rates.get(amt, 60 if amt and 60000 <= amt <= 100000 else None)

        if rate:
            yrs = simpledialog.askfloat("Loan", "Duration in years:")
            if yrs:
                total_due = amt + (amt * rate * yrs / 100)
                user_ref["balance"] += amt
                user_ref["loan"] = {"unpaid": total_due, "monthly": total_due / (yrs * 12)}
                log_transaction(username, "LOAN TAKEN", amt, f"({rate}% interest)")
                save_data()
        else:
            messagebox.showerror("Error", "Unsupported loan amount.")

    # 5. PAY LOAN
    elif choice == "5":
        if user_ref["loan"]["unpaid"] <= 0:
            messagebox.showinfo("Loan", "No unpaid loans.")
        else:
            amt = simpledialog.askfloat("Pay Loan", f"Debt: ₱{user_ref['loan']['unpaid']:.2f}\nEnter Payment:")
            if amt and amt <= user_ref["balance"]:
                payment = min(amt, user_ref["loan"]["unpaid"])
                user_ref["balance"] -= payment
                user_ref["loan"]["unpaid"] -= payment
                log_transaction(username, "LOAN PAYMENT", payment)
                save_data()
            else:
                messagebox.showerror("Error", "Insufficient balance.")

    # 6. TRANSFER MONEY
    elif choice == "6":
        target = simpledialog.askstring("Transfer", "Recipient Name:")
        if target in users and target != username:
            amt = simpledialog.askfloat("Transfer", f"Amount to {target}:")
            if amt and amt <= user_ref["balance"]:
                user_ref["balance"] -= amt
                users[target]["balance"] += amt
                log_transaction(username, f"TRANSFER TO {target}", amt)
                log_transaction(target, f"RECEIVED FROM {username}", amt)
                save_data()
                messagebox.showinfo("Success", "Transfer Complete")
            else:
                messagebox.showerror("Error", "Insufficient funds.")
        else:
            messagebox.showerror("Error", "User not found.")

    # 7. EXPORT PERSONAL LOG
    elif choice == "7":
        export_user_log(username)

    # EXIT
    elif choice == "8":
        save_data()
        messagebox.showinfo("Exit", "Thank you for banking with us!")
        break

    else:
        messagebox.showwarning("Invalid", "Choose a valid option.")
