import tkinter as tk
from tkinter import ttk, messagebox, simpledialog, filedialog
import sqlite3, csv, shutil, hashlib, json
from datetime import datetime
from pathlib import Path

APP_DIR = Path.home() / "MandoCenterManager"
APP_DIR.mkdir(exist_ok=True)
DB = APP_DIR / "center.db"

def now():
    return datetime.now().isoformat(sep=" ", timespec="seconds")

def connect():
    c=sqlite3.connect(DB)
    c.row_factory=sqlite3.Row
    c.execute("PRAGMA foreign_keys=ON")
    return c

def init_db():
    with connect() as c:
        c.executescript("""
        CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, username TEXT UNIQUE, password_hash TEXT, role TEXT);
        CREATE TABLE IF NOT EXISTS students(id INTEGER PRIMARY KEY, barcode TEXT UNIQUE, name TEXT NOT NULL, phone TEXT DEFAULT '', grade TEXT DEFAULT '', plan TEXT DEFAULT 'per_lesson', lesson_fee REAL DEFAULT 0, monthly_fee REAL DEFAULT 0, active INTEGER DEFAULT 1, created_at TEXT);
        CREATE TABLE IF NOT EXISTS groups_(id INTEGER PRIMARY KEY, name TEXT UNIQUE NOT NULL, subject TEXT DEFAULT '', instructor TEXT DEFAULT '', schedule TEXT DEFAULT '');
        CREATE TABLE IF NOT EXISTS enrollments(student_id INTEGER, group_id INTEGER, PRIMARY KEY(student_id,group_id), FOREIGN KEY(student_id) REFERENCES students(id), FOREIGN KEY(group_id) REFERENCES groups_(id));
        CREATE TABLE IF NOT EXISTS sessions(id INTEGER PRIMARY KEY, group_id INTEGER, session_date TEXT, start_time TEXT DEFAULT '', FOREIGN KEY(group_id) REFERENCES groups_(id));
        CREATE TABLE IF NOT EXISTS attendance(id INTEGER PRIMARY KEY, student_id INTEGER, session_id INTEGER, status TEXT, scanned_at TEXT, UNIQUE(student_id,session_id), FOREIGN KEY(student_id) REFERENCES students(id), FOREIGN KEY(session_id) REFERENCES sessions(id));
        CREATE TABLE IF NOT EXISTS transactions(id INTEGER PRIMARY KEY, student_id INTEGER, amount REAL, kind TEXT, method TEXT, note TEXT DEFAULT '', created_at TEXT, user TEXT, FOREIGN KEY(student_id) REFERENCES students(id));
        CREATE TABLE IF NOT EXISTS audit(id INTEGER PRIMARY KEY, at TEXT, username TEXT, action TEXT, entity TEXT, entity_id TEXT, details TEXT, prev_hash TEXT, entry_hash TEXT);
        """)
        if not c.execute("SELECT 1 FROM users").fetchone():
            c.execute("INSERT INTO users(username,password_hash,role) VALUES(?,?,?)",("admin",hashlib.sha256(b"ChangeMe123!").hexdigest(),"admin"))

def audit(user, action, entity, entity_id, details):
    with connect() as c:
        prev=c.execute("SELECT entry_hash FROM audit ORDER BY id DESC LIMIT 1").fetchone()
        prev_hash=prev["entry_hash"] if prev else "GENESIS"
        payload=f"{now()}|{user}|{action}|{entity}|{entity_id}|{details}|{prev_hash}"
        h=hashlib.sha256(payload.encode()).hexdigest()
        c.execute("INSERT INTO audit(at,username,action,entity,entity_id,details,prev_hash,entry_hash) VALUES(?,?,?,?,?,?,?,?)",
                  (now(),user,action,entity,str(entity_id),details,prev_hash,h))

class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Mando Center Manager | إدارة السنتر")
        self.geometry("1120x720")
        self.user=None
        self.login()

    def login(self):
        for w in self.winfo_children(): w.destroy()
        f=ttk.Frame(self,padding=30); f.pack(expand=True)
        ttk.Label(f,text="Mando Center Manager",font=("Arial",22,"bold")).grid(row=0,column=0,columnspan=2,pady=12)
        ttk.Label(f,text="اسم المستخدم").grid(row=1,column=0,sticky="e",pady=5)
        u=ttk.Entry(f); u.insert(0,"admin"); u.grid(row=1,column=1)
        ttk.Label(f,text="كلمة المرور").grid(row=2,column=0,sticky="e",pady=5)
        p=ttk.Entry(f,show="*"); p.grid(row=2,column=1)
        def go():
            row=connect().execute("SELECT * FROM users WHERE username=? AND password_hash=?",(u.get().strip(),hashlib.sha256(p.get().encode()).hexdigest())).fetchone()
            if row:
                self.user=row["username"]; self.main()
            else: messagebox.showerror("خطأ","بيانات الدخول غير صحيحة")
        ttk.Button(f,text="دخول",command=go).grid(row=3,column=0,columnspan=2,pady=12)
        ttk.Label(f,text="الدخول الأول: admin / ChangeMe123! — غيّرها فورًا من إدارة المستخدمين (قيد التطوير).").grid(row=4,column=0,columnspan=2,pady=8)

    def main(self):
        for w in self.winfo_children(): w.destroy()
        top=ttk.Frame(self,padding=10); top.pack(fill="x")
        ttk.Label(top,text="Mando Center Manager",font=("Arial",18,"bold")).pack(side="left")
        ttk.Label(top,text=f"المستخدم: {self.user}").pack(side="right")
        tabs=ttk.Notebook(self); tabs.pack(fill="both",expand=True,padx=10,pady=10)
        self.student_tab(tabs); self.group_tab(tabs); self.attendance_tab(tabs); self.payment_tab(tabs); self.audit_tab(tabs); self.tools_tab(tabs)

    def tree(self,parent,cols,heads):
        t=ttk.Treeview(parent,columns=cols,show="headings",height=17)
        for col,head in zip(cols,heads):
            t.heading(col,text=head); t.column(col,width=130,anchor="center")
        t.pack(fill="both",expand=True,pady=8)
        return t

    def student_tab(self,tabs):
        f=ttk.Frame(tabs,padding=10); tabs.add(f,text="الطلاب")
        bar=ttk.Frame(f); bar.pack(fill="x")
        self.stree=self.tree(f,("id","barcode","name","phone","grade","plan","fee","active"),("ID","Barcode","اسم الطالب","الهاتف","الصف","النظام","قيمة الحصة","نشط"))
        ttk.Button(bar,text="إضافة طالب",command=self.add_student).pack(side="left",padx=3)
        ttk.Button(bar,text="تحديث",command=self.load_students).pack(side="left",padx=3)
        ttk.Button(bar,text="أرشفة المحدد",command=self.archive_student).pack(side="left",padx=3)
        self.load_students()
    def load_students(self):
        if not hasattr(self,"stree"): return
        self.stree.delete(*self.stree.get_children())
        for r in connect().execute("SELECT * FROM students ORDER BY id DESC"):
            self.stree.insert("", "end",values=(r["id"],r["barcode"],r["name"],r["phone"],r["grade"],r["plan"],r["lesson_fee"],"نعم" if r["active"] else "لا"))
    def add_student(self):
        d=tk.Toplevel(self); d.title("إضافة طالب"); d.grab_set()
        fields=[("الاسم","name"),("Barcode / ID","barcode"),("الهاتف","phone"),("الصف","grade"),("النظام: per_lesson أو monthly","plan"),("سعر الحصة","lesson_fee"),("الاشتراك الشهري","monthly_fee")]
        entries={}
        for i,(label,key) in enumerate(fields):
            ttk.Label(d,text=label).grid(row=i,column=0,padx=8,pady=4,sticky="e")
            e=ttk.Entry(d,width=35); e.grid(row=i,column=1,padx=8,pady=4); entries[key]=e
        entries["plan"].insert(0,"per_lesson"); entries["lesson_fee"].insert(0,"0"); entries["monthly_fee"].insert(0,"0")
        def save():
            try:
                vals={k:e.get().strip() for k,e in entries.items()}
                if not vals["name"] or not vals["barcode"]: raise ValueError("الاسم والباركود مطلوبان")
                if vals["plan"] not in ("per_lesson","monthly"): raise ValueError("النظام يجب أن يكون per_lesson أو monthly")
                with connect() as c:
                    cur=c.execute("INSERT INTO students(barcode,name,phone,grade,plan,lesson_fee,monthly_fee,created_at) VALUES(?,?,?,?,?,?,?,?)",
                    (vals["barcode"],vals["name"],vals["phone"],vals["grade"],vals["plan"],float(vals["lesson_fee"] or 0),float(vals["monthly_fee"] or 0),now()))
                    sid=cur.lastrowid
                audit(self.user,"CREATE","student",sid,json.dumps(vals,ensure_ascii=False))
                self.load_students(); d.destroy()
            except Exception as e: messagebox.showerror("تعذر الحفظ",str(e))
        ttk.Button(d,text="حفظ الطالب",command=save).grid(row=len(fields),column=0,columnspan=2,pady=10)
    def archive_student(self):
        sel=self.stree.selection()
        if not sel: return
        sid=self.stree.item(sel[0])["values"][0]
        if messagebox.askyesno("تأكيد","أرشفة الطالب؟ لن يتم حذف سجله."):
            with connect() as c: c.execute("UPDATE students SET active=0 WHERE id=?",(sid,))
            audit(self.user,"ARCHIVE","student",sid,"active=0"); self.load_students()

    def group_tab(self,tabs):
        f=ttk.Frame(tabs,padding=10); tabs.add(f,text="المجموعات")
        bar=ttk.Frame(f); bar.pack(fill="x")
        self.gtree=self.tree(f,("id","name","subject","instructor","schedule"),("ID","اسم المجموعة","المادة","المدرس","الجدول"))
        ttk.Button(bar,text="إضافة مجموعة",command=self.add_group).pack(side="left",padx=3)
        ttk.Button(bar,text="تحديث",command=self.load_groups).pack(side="left",padx=3)
        ttk.Button(bar,text="تسجيل طالب بالمجموعة",command=self.enroll).pack(side="left",padx=3)
        self.load_groups()
    def load_groups(self):
        self.gtree.delete(*self.gtree.get_children())
        for r in connect().execute("SELECT * FROM groups_ ORDER BY name"):
            self.gtree.insert("", "end",values=tuple(r[k] for k in ("id","name","subject","instructor","schedule")))
    def add_group(self):
        vals=self.simple_form("إضافة مجموعة",[("اسم المجموعة","name"),("المادة","subject"),("المدرس","instructor"),("المواعيد","schedule")])
        if not vals:return
        try:
            with connect() as c: cur=c.execute("INSERT INTO groups_(name,subject,instructor,schedule) VALUES(?,?,?,?)",tuple(vals.values()))
            audit(self.user,"CREATE","group",cur.lastrowid,str(vals)); self.load_groups()
        except Exception as e: messagebox.showerror("خطأ",str(e))
    def simple_form(self,title,fields):
        d=tk.Toplevel(self); d.title(title); d.grab_set(); es={}
        for i,(lab,key) in enumerate(fields):
            ttk.Label(d,text=lab).grid(row=i,column=0,padx=8,pady=4)
            e=ttk.Entry(d,width=32); e.grid(row=i,column=1,padx=8,pady=4); es[key]=e
        out={}
        def save():
            out.update({k:e.get().strip() for k,e in es.items()}); d.destroy()
        ttk.Button(d,text="حفظ",command=save).grid(row=len(fields),column=0,columnspan=2,pady=8)
        d.wait_window()
        return out or None
    def enroll(self):
        sid=simpledialog.askinteger("تسجيل طالب","أدخل Student ID")
        gid=simpledialog.askinteger("تسجيل طالب","أدخل Group ID")
        if not sid or not gid:return
        try:
            with connect() as c:c.execute("INSERT INTO enrollments(student_id,group_id) VALUES(?,?)",(sid,gid))
            audit(self.user,"ENROLL","enrollment",f"{sid}:{gid}","")
        except Exception as e:messagebox.showerror("خطأ",str(e))

    def attendance_tab(self,tabs):
        f=ttk.Frame(tabs,padding=10); tabs.add(f,text="الحضور بالباركود")
        ttk.Label(f,text="امسح كارت الطالب في الخانة — جهاز USB غالبًا يعمل كلوحة مفاتيح").pack(anchor="w")
        bar=ttk.Frame(f); bar.pack(fill="x",pady=8)
        self.scan=ttk.Entry(bar,font=("Arial",16)); self.scan.pack(side="left",fill="x",expand=True); self.scan.bind("<Return>",self.scan_attendance)
        ttk.Button(bar,text="تسجيل حضور",command=lambda:self.scan_attendance()).pack(side="left",padx=5)
        ttk.Label(f,text="اختر المجموعة/الحصة الحالية قبل التسجيل.").pack(anchor="w")
        self.session_group=tk.StringVar()
        self.session_combo=ttk.Combobox(f,textvariable=self.session_group,state="readonly",width=50); self.session_combo.pack(anchor="w",pady=5)
        ttk.Button(f,text="إنشاء حصة اليوم للمجموعة المحددة",command=self.create_session).pack(anchor="w",pady=5)
        self.atree=self.tree(f,("time","student","barcode","status","group"),("الوقت","الطالب","Barcode","الحالة","المجموعة"))
        self.load_session_groups(); self.load_attendance()
    def load_session_groups(self):
        rows=connect().execute("SELECT id,name FROM groups_ ORDER BY name").fetchall()
        self.group_map={f'{r["id"]} - {r["name"]}':r["id"] for r in rows}
        self.session_combo["values"]=list(self.group_map)
        if rows:self.session_combo.current(0)
    def create_session(self):
        label=self.session_group.get()
        if not label: messagebox.showwarning("تنبيه","أضف مجموعة أولًا"); return
        gid=self.group_map[label]
        with connect() as c: cur=c.execute("INSERT INTO sessions(group_id,session_date,start_time) VALUES(?,?,?)",(gid,datetime.now().strftime("%Y-%m-%d"),datetime.now().strftime("%H:%M")))
        self.current_session=cur.lastrowid
        audit(self.user,"CREATE","session",self.current_session,label)
        messagebox.showinfo("تم","تم إنشاء حصة اليوم. امسح كروت الطلاب.")
    def scan_attendance(self,event=None):
        code=self.scan.get().strip()
        self.scan.delete(0,"end")
        if not code:return
        if not getattr(self,"current_session",None):
            messagebox.showwarning("لا توجد حصة","أنشئ حصة اليوم أولًا"); return
        with connect() as c:
            s=c.execute("SELECT * FROM students WHERE barcode=? AND active=1",(code,)).fetchone()
            if not s: messagebox.showerror("غير معروف","الباركود غير مسجل أو الطالب غير نشط"); return
            sess=c.execute("SELECT group_id FROM sessions WHERE id=?",(self.current_session,)).fetchone()
            enrolled=c.execute("SELECT 1 FROM enrollments WHERE student_id=? AND group_id=?",(s["id"],sess["group_id"])).fetchone()
            if not enrolled:
                messagebox.showwarning("تنبيه","الطالب غير مسجل في المجموعة المحددة"); return
            try:c.execute("INSERT INTO attendance(student_id,session_id,status,scanned_at) VALUES(?,?,?,?)",(s["id"],self.current_session,"present",now()))
            except sqlite3.IntegrityError:messagebox.showwarning("مكرر","تم تسجيل حضور الطالب لهذه الحصة بالفعل"); return
            if s["plan"]=="per_lesson":
                c.execute("INSERT INTO transactions(student_id,amount,kind,method,note,created_at,user) VALUES(?,?,?,?,?,?,?)",(s["id"],-float(s["lesson_fee"]),"lesson_charge","system",f"خصم حصة session={self.current_session}",now(),self.user))
        audit(self.user,"ATTENDANCE","student",s["id"],f"session={self.current_session};barcode={code}")
        if s["plan"]=="per_lesson":audit(self.user,"LESSON_CHARGE","student",s["id"],f"fee={s['lesson_fee']};session={self.current_session}")
        self.load_attendance(); messagebox.showinfo("تم الحضور",f"تم تسجيل حضور: {s['name']}")
    def load_attendance(self):
        if not hasattr(self,"atree"):return
        self.atree.delete(*self.atree.get_children())
        for r in connect().execute("""SELECT a.scanned_at,s.name,s.barcode,a.status,g.name groupname
        FROM attendance a JOIN students s ON s.id=a.student_id JOIN sessions se ON se.id=a.session_id JOIN groups_ g ON g.id=se.group_id ORDER BY a.id DESC LIMIT 500"""):
            self.atree.insert("","end",values=(r["scanned_at"],r["name"],r["barcode"],r["status"],r["groupname"]))

    def payment_tab(self,tabs):
        f=ttk.Frame(tabs,padding=10); tabs.add(f,text="المدفوعات")
        bar=ttk.Frame(f); bar.pack(fill="x")
        self.ptree=self.tree(f,("id","student","amount","kind","method","date","user"),("رقم","Student ID","المبلغ (+قبض / -رسوم)","النوع","الطريقة","التاريخ","المستخدم"))
        ttk.Button(bar,text="تسجيل دفعة",command=self.add_payment).pack(side="left",padx=3)
        ttk.Button(bar,text="تحديث",command=self.load_payments).pack(side="left",padx=3)
        ttk.Button(bar,text="كشف حساب طالب",command=self.statement).pack(side="left",padx=3)
        self.load_payments()
    def load_payments(self):
        self.ptree.delete(*self.ptree.get_children())
        for r in connect().execute("SELECT * FROM transactions ORDER BY id DESC LIMIT 1000"):
            self.ptree.insert("","end",values=(r["id"],r["student_id"],r["amount"],r["kind"],r["method"],r["created_at"],r["user"]))
    def add_payment(self):
        vals=self.simple_form("تسجيل دفعة",[("Student ID","student_id"),("المبلغ","amount"),("الطريقة","method"),("ملاحظات","note")])
        if not vals:return
        try:
            sid=int(vals["student_id"]); amount=float(vals["amount"])
            with connect() as c: cur=c.execute("INSERT INTO transactions(student_id,amount,kind,method,note,created_at,user) VALUES(?,?,?,?,?,?,?)",(sid,amount,"payment",vals["method"] or "cash",vals["note"],now(),self.user))
            audit(self.user,"PAYMENT","transaction",cur.lastrowid,json.dumps(vals,ensure_ascii=False)); self.load_payments()
        except Exception as e:messagebox.showerror("خطأ",str(e))
    def statement(self):
        sid=simpledialog.askinteger("كشف حساب","Student ID")
        if not sid:return
        rows=connect().execute("SELECT amount,kind,method,note,created_at FROM transactions WHERE student_id=? ORDER BY id",(sid,)).fetchall()
        if not rows:messagebox.showinfo("كشف حساب","لا توجد حركات");return
        total=sum(r["amount"] for r in rows)
        txt="\n".join(f'{r["created_at"]} | {r["kind"]} | {r["amount"]:.2f} | {r["method"]} | {r["note"]}' for r in rows)
        messagebox.showinfo(f"كشف حساب الطالب {sid}",txt+f"\n\nصافي الرصيد (الموجب = رصيد لصالح الطالب): {total:.2f}")

    def audit_tab(self,tabs):
        f=ttk.Frame(tabs,padding=10); tabs.add(f,text="سجل العمليات")
        self.audit_tree=self.tree(f,("id","at","user","action","entity","entity_id","details","hash"),("ID","التاريخ","المستخدم","العملية","الكيان","المعرف","التفاصيل","Hash"))
        ttk.Button(f,text="تحديث السجل",command=self.load_audit).pack(anchor="w")
        self.load_audit()
    def load_audit(self):
        self.audit_tree.delete(*self.audit_tree.get_children())
        for r in connect().execute("SELECT * FROM audit ORDER BY id DESC LIMIT 1000"):
            self.audit_tree.insert("","end",values=(r["id"],r["at"],r["username"],r["action"],r["entity"],r["entity_id"],r["details"],r["entry_hash"][:16]))
    def tools_tab(self,tabs):
        f=ttk.Frame(tabs,padding=15); tabs.add(f,text="النسخ الاحتياطي والتصدير")
        ttk.Button(f,text="إنشاء نسخة احتياطية",command=self.backup).pack(anchor="w",pady=6)
        ttk.Button(f,text="تصدير الطلاب CSV",command=self.export_students).pack(anchor="w",pady=6)
        ttk.Button(f,text="تصدير الحضور CSV",command=self.export_attendance).pack(anchor="w",pady=6)
        ttk.Label(f,text=f"مسار قاعدة البيانات: {DB}").pack(anchor="w",pady=10)
        ttk.Label(f,text="تنبيه: هذه نسخة MVP أولية. راجع README قبل استخدامها ببيانات حقيقية.").pack(anchor="w")
    def backup(self):
        dest=filedialog.asksaveasfilename(defaultextension=".db",filetypes=[("SQLite database","*.db")],initialfile=f"center_backup_{datetime.now():%Y%m%d_%H%M}.db")
        if dest:
            with connect() as c:
                target=sqlite3.connect(dest); c.backup(target); target.close()
            audit(self.user,"BACKUP","database",dest,"SQLite backup")
            messagebox.showinfo("تم","تم إنشاء النسخة الاحتياطية")
    def export_students(self):
        dest=filedialog.asksaveasfilename(defaultextension=".csv",filetypes=[("CSV","*.csv")])
        if not dest:return
        rows=connect().execute("SELECT * FROM students").fetchall()
        with open(dest,"w",newline="",encoding="utf-8-sig") as f:
            w=csv.writer(f); w.writerow(rows[0].keys() if rows else ["id"]); w.writerows([tuple(r) for r in rows])
        audit(self.user,"EXPORT","students",dest,"CSV")
    def export_attendance(self):
        dest=filedialog.asksaveasfilename(defaultextension=".csv",filetypes=[("CSV","*.csv")])
        if not dest:return
        rows=connect().execute("""SELECT s.name,s.barcode,g.name group_name,se.session_date,a.scanned_at,a.status FROM attendance a JOIN students s ON s.id=a.student_id JOIN sessions se ON se.id=a.session_id JOIN groups_ g ON g.id=se.group_id""").fetchall()
        with open(dest,"w",newline="",encoding="utf-8-sig") as f:
            w=csv.writer(f); w.writerow(rows[0].keys() if rows else ["name"]); w.writerows([tuple(r) for r in rows])
        audit(self.user,"EXPORT","attendance",dest,"CSV")

if __name__=="__main__":
    init_db()
    App().mainloop()
