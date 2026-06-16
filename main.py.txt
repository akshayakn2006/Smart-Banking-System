import tkinter as tk
from tkinter import ttk, messagebox
from PIL import Image, ImageTk
import mysql.connector
from datetime import datetime

# ---------- DB ----------
conn = mysql.connector.connect(
    host="localhost",
    user="root",
    password="YOUR_PASSWORD",
    database="bank_db"
)
cursor = conn.cursor()

# ---------- COLORS ----------
BG_MAIN = "#f3e5f5"
SIDEBAR = "#4a148c"
BTN = "#7b1fa2"
HOVER = "#9c27b0"
CARD = "#ede7f6"

# ---------- TRANSACTION ----------
def add_transaction(acc_no, t_type, amount):
    try:
        today = datetime.now().date()
        cursor.execute(
            "INSERT INTO transactions (acc_no, type, amount, date) VALUES (%s,%s,%s,%s)",
            (int(acc_no), t_type, int(amount), today)
        )
        conn.commit()
    except Exception as e:
        print("Transaction Error:", e)

# ---------- DASHBOARD ----------
def load_dashboard_ui():
    for widget in main.winfo_children():
        widget.destroy()

    tk.Label(main, text="Dashboard",
             font=("Segoe UI", 24, "bold"),
             bg=BG_MAIN, fg="#4a148c").pack(pady=20)

    card_frame = tk.Frame(main, bg=BG_MAIN)
    card_frame.pack(pady=20)

    def create_card(title, value):
        frame = tk.Frame(card_frame, bg=CARD, width=200, height=120)
        frame.pack(side="left", padx=20)

        tk.Label(frame, text=title,
                 font=("Segoe UI", 12, "bold"),
                 bg=CARD, fg="#4a148c").pack(pady=10)

        tk.Label(frame, text=value,
                 font=("Segoe UI", 20, "bold"),
                 bg=CARD).pack()

    try:
        cursor.execute("SELECT COUNT(*) FROM accounts")
        total_accounts = cursor.fetchone()[0]

        cursor.execute("SELECT SUM(balance) FROM accounts")
        total_balance = cursor.fetchone()[0] or 0

        cursor.execute("SELECT COUNT(*) FROM transactions")
        total_trans = cursor.fetchone()[0]
    except:
        total_accounts = total_balance = total_trans = 0

    create_card("Total Accounts", total_accounts)
    create_card("Total Balance", total_balance)
    create_card("Transactions", total_trans)

# ---------- CUSTOMER ----------
def clear_fields():
    acc_entry.delete(0, tk.END)
    name_entry.delete(0, tk.END)
    bal_entry.delete(0, tk.END)

def add_account():
    try:
        cursor.execute("INSERT INTO accounts VALUES (%s,%s,%s)",
                       (int(acc_entry.get()), name_entry.get(), int(bal_entry.get())))
        conn.commit()
        show_data()
        messagebox.showinfo("Success", "Account Added!")
    except Exception as e:
        messagebox.showerror("Error", str(e))

def update_account():
    cursor.execute("UPDATE accounts SET name=%s, balance=%s WHERE acc_no=%s",
                   (name_entry.get(), int(bal_entry.get()), int(acc_entry.get())))
    conn.commit()
    show_data()
    messagebox.showinfo("Updated", "Account Updated!")

def delete_account():
    cursor.execute("DELETE FROM accounts WHERE acc_no=%s",
                   (int(acc_entry.get()),))
    conn.commit()
    show_data()
    messagebox.showinfo("Deleted", "Account Deleted!")

def show_data():
    cursor.execute("SELECT * FROM accounts")
    rows = cursor.fetchall()
    table.delete(*table.get_children())

    for i, row in enumerate(rows):
        tag = "even" if i % 2 == 0 else "odd"
        table.insert("", tk.END, values=row, tags=(tag,))

    table.tag_configure("even", background="#f8f5ff")
    table.tag_configure("odd", background="#ede7f6")

def select_data(event):
    selected = table.focus()
    values = table.item(selected, "values")
    if values:
        acc_entry.delete(0, tk.END)
        acc_entry.insert(0, values[0])
        name_entry.delete(0, tk.END)
        name_entry.insert(0, values[1])
        bal_entry.delete(0, tk.END)
        bal_entry.insert(0, values[2])

# ---------- ACCOUNT ----------
def load_account_ui():
    for widget in main.winfo_children():
        widget.destroy()

    tk.Label(main, text="Account Operations",
             font=("Segoe UI", 22, "bold"),
             bg=BG_MAIN, fg="#4a148c").pack(pady=10)

    form = tk.Frame(main, bg=CARD)
    form.pack(pady=20, ipadx=20, ipady=20)

    tk.Label(form, text="Account No", bg=CARD).grid(row=0, column=0, pady=10)
    acc_no = tk.Entry(form)
    acc_no.grid(row=0, column=1)

    tk.Label(form, text="Amount", bg=CARD).grid(row=1, column=0, pady=10)
    amount = tk.Entry(form)
    amount.grid(row=1, column=1)

    def deposit():
        try:
            if acc_no.get() == "" or amount.get() == "":
                messagebox.showerror("Error", "Enter all fields")
                return

            cursor.execute(
                "UPDATE accounts SET balance = balance + %s WHERE acc_no = %s",
                (int(amount.get()), int(acc_no.get()))
            )
            conn.commit()

            add_transaction(acc_no.get(), "Deposit", amount.get())
            messagebox.showinfo("Success", "Amount Deposited!")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def withdraw():
        try:
            if acc_no.get() == "" or amount.get() == "":
                messagebox.showerror("Error", "Enter all fields")
                return

            cursor.execute("SELECT balance FROM accounts WHERE acc_no=%s",
                           (int(acc_no.get()),))
            bal = cursor.fetchone()

            if bal and bal[0] >= int(amount.get()):
                cursor.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE acc_no = %s",
                    (int(amount.get()), int(acc_no.get()))
                )
                conn.commit()

                add_transaction(acc_no.get(), "Withdraw", amount.get())
                messagebox.showinfo("Success", "Amount Withdrawn!")
            else:
                messagebox.showerror("Error", "Insufficient Balance!")
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def check_balance():
        cursor.execute("SELECT balance FROM accounts WHERE acc_no=%s",
                       (int(acc_no.get()),))
        bal = cursor.fetchone()

        if bal:
            messagebox.showinfo("Balance", f"Current Balance: {bal[0]}")
        else:
            messagebox.showerror("Error", "Account Not Found!")

    btn_frame = tk.Frame(form, bg=CARD)
    btn_frame.grid(row=2, columnspan=2, pady=15)

    tk.Button(btn_frame, text="Deposit", bg="#6a1b9a", fg="white",
              width=10, command=deposit).pack(side="left", padx=5)

    tk.Button(btn_frame, text="Withdraw", bg="#8e24aa", fg="white",
              width=10, command=withdraw).pack(side="left", padx=5)

    tk.Button(btn_frame, text="Check Balance", bg="#9575cd", fg="white",
              width=15, command=check_balance).pack(side="left", padx=5)

# ---------- TRANSACTION UI ----------
def load_transaction_ui():
    for widget in main.winfo_children():
        widget.destroy()

    tk.Label(main, text="Transaction History",
             font=("Segoe UI", 22, "bold"),
             bg=BG_MAIN, fg="#4a148c").pack(pady=10)

    form = tk.Frame(main, bg=CARD)
    form.pack(pady=10)

    tk.Label(form, text="Account No", bg=CARD).grid(row=0, column=0, padx=10, pady=10)
    acc_no = tk.Entry(form)
    acc_no.grid(row=0, column=1)

    table_frame = tk.Frame(main)
    table_frame.pack(fill="both", expand=True, pady=10)

    trans_table = ttk.Treeview(
        table_frame,
        columns=("type", "amount", "date"),
        show="headings"
    )

    trans_table.heading("type", text="Type")
    trans_table.heading("amount", text="Amount")
    trans_table.heading("date", text="Date")

    trans_table.pack(fill="both", expand=True)

    def load_transactions():
        try:
            if acc_no.get() == "":
                messagebox.showerror("Error", "Enter Account Number")
                return

            cursor.execute(
                "SELECT type, amount, date FROM transactions WHERE acc_no=%s ORDER BY date DESC",
                (int(acc_no.get()),)
            )

            rows = cursor.fetchall()
            trans_table.delete(*trans_table.get_children())

            for row in rows:
                clean_date = str(row[2])[:10]
                trans_table.insert("", tk.END, values=(row[0], row[1], clean_date))

        except Exception as e:
            messagebox.showerror("Error", str(e))

    tk.Button(form, text="Show Transactions",
              bg="#6a1b9a", fg="white",
              command=load_transactions).grid(row=1, columnspan=2, pady=10)

# ---------- NAVIGATION ----------
def show_section(section):
    for widget in main.winfo_children():
        widget.destroy()

    if section == "Dashboard":
        load_dashboard_ui()
    elif section == "Customer":
        load_customer_ui()
    elif section == "Account":
        load_account_ui()
    elif section == "Transaction":
        load_transaction_ui()

# ---------- HOVER ----------
def on_enter(e): e.widget['bg'] = HOVER
def on_leave(e): e.widget['bg'] = BTN

# ---------- WINDOW ----------
root = tk.Tk()
root.title("Bank Dashboard")
root.geometry("1100x650")
root.config(bg=BG_MAIN)

# ---------- SIDEBAR ----------
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.pack(side="left", fill="y")

try:
    logo_img = Image.open("images/logo.png").resize((120, 120))
    logo = ImageTk.PhotoImage(logo_img)
    tk.Label(sidebar, image=logo, bg=SIDEBAR).pack(pady=20)
except:
    tk.Label(sidebar, text="🏦 BANK", fg="white", bg=SIDEBAR,
             font=("Segoe UI", 16, "bold")).pack(pady=20)

def menu_btn(text):
    btn = tk.Button(
        sidebar,
        text=text,
        bg=BTN,
        fg="white",
        font=("Segoe UI", 11, "bold"),
        width=18,
        bd=0,
        pady=10,
        cursor="hand2",
        command=lambda: show_section(text)
    )
    btn.pack(pady=6)
    btn.bind("<Enter>", on_enter)
    btn.bind("<Leave>", on_leave)

# ✅ ORDER FIXED
menu_btn("Customer")
menu_btn("Account")
menu_btn("Transaction")
menu_btn("Dashboard")

# ---------- MAIN ----------
main = tk.Frame(root, bg=BG_MAIN)
main.pack(side="right", fill="both", expand=True, padx=20, pady=20)

# ---------- CUSTOMER UI ----------
def load_customer_ui():
    global acc_entry, name_entry, bal_entry, table

    tk.Label(main, text="SMART BANKING",
             font=("Segoe UI", 22, "bold"),
             bg=BG_MAIN, fg="#4a148c").pack(pady=10)

    form = tk.Frame(main, bg=CARD)
    form.pack(pady=10, ipadx=20, ipady=20)

    def styled_entry(parent):
        return tk.Entry(parent, bd=1, relief="solid", font=("Segoe UI", 10))

    tk.Label(form, text="Account No", bg=CARD).grid(row=0, column=0, padx=10, pady=8)
    acc_entry = styled_entry(form)
    acc_entry.grid(row=0, column=1)

    tk.Label(form, text="Name", bg=CARD).grid(row=1, column=0, padx=10, pady=8)
    name_entry = styled_entry(form)
    name_entry.grid(row=1, column=1)

    tk.Label(form, text="Balance", bg=CARD).grid(row=2, column=0, padx=10, pady=8)
    bal_entry = styled_entry(form)
    bal_entry.grid(row=2, column=1)

    btn_frame = tk.Frame(form, bg=CARD)
    btn_frame.grid(row=3, column=0, columnspan=2, pady=15)

    def action_btn(text, color, cmd):
        tk.Button(btn_frame, text=text, bg=color, fg="white",
                  width=10, bd=0, pady=6, command=cmd).pack(side="left", padx=5)

    action_btn("Add", "#6a1b9a", add_account)
    action_btn("Update", "#8e24aa", update_account)
    action_btn("Delete", "#ab47bc", delete_account)
    action_btn("Clear", "#9575cd", clear_fields)

    style = ttk.Style()
    style.theme_use("default")

    table = ttk.Treeview(main, columns=("acc", "name", "bal"), show="headings")

    table.heading("acc", text="Account No")
    table.heading("name", text="Name")
    table.heading("bal", text="Balance")

    table.pack(fill="both", expand=True, pady=10)
    table.bind("<ButtonRelease-1>", select_data)

    show_data()

# ---------- START ----------
show_section("Customer")
root.mainloop()
