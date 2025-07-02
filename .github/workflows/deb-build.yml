import sys
import os
import shutil
import subprocess
import platform
import smtplib
import json
from pathlib import Path
from email.mime.text import MIMEText
from PyQt5.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QCheckBox, QHBoxLayout,
    QPushButton, QTextEdit, QLabel, QMessageBox, QLineEdit,
    QComboBox, QProgressBar
)

class CleanerApp(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Cross-Platform Cleaner")
        self.resize(600, 750)
        layout = QVBoxLayout()

        # Progress bar
        self.progress = QProgressBar()
        layout.addWidget(self.progress)

        # Cleaning options
        self.options = {
            'Temp files': self.clean_temp,
            'Browser cache': self.clean_browser_cache,
            'Trash': self.clean_trash,
            'System logs': self.clean_system_logs,
            'Apt cache': self.clean_apt_cache,
            'Pip cache': self.clean_pip_cache,
            'Thumbnail cache': self.clean_thumbnails,
            'Docker prune': self.clean_docker,
            'Flatpak cache': self.clean_flatpak_cache,
            'Snap cache': self.clean_snap_cache,
            'Autoremove packages': self.clean_autoremove,
            'Defragmentation': self.clean_defrag_all,
            'Registry clean': self.clean_registry
        }

        self.checkboxes = {}
        for label in self.options:
            cb = QCheckBox(label)
            layout.addWidget(cb)
            self.checkboxes[label] = cb

        # Scheduler
        hbox_sched = QHBoxLayout()
        self.cb_schedule = QCheckBox("Добавить в планировщик")
        self.schedule_combo = QComboBox()
        self.schedule_combo.addItems(["Ежедневно", "Еженедельно", "Ежемесячно"])
        hbox_sched.addWidget(self.cb_schedule)
        hbox_sched.addWidget(self.schedule_combo)
        layout.addLayout(hbox_sched)

        # Email & Slack
        layout.addWidget(QLabel("Email для уведомлений (SMTP vars должны быть настроены):"))
        self.input_email = QLineEdit()
        layout.addWidget(self.input_email)
        layout.addWidget(QLabel("Slack webhook URL:"))
        self.input_slack = QLineEdit()
        layout.addWidget(self.input_slack)

        # Custom paths
        hbox = QHBoxLayout()
        self.custom_path_input = QLineEdit()
        self.custom_path_input.setPlaceholderText("Дополнительный путь для очистки")
        self.btn_add = QPushButton("Добавить путь")
        self.btn_add.clicked.connect(self.add_custom_path)
        hbox.addWidget(self.custom_path_input)
        hbox.addWidget(self.btn_add)
        layout.addLayout(hbox)
        self.custom_paths = []

        # Run button & log
        self.btn_clean = QPushButton("Запустить очистку")
        self.btn_clean.clicked.connect(self.run_clean)
        layout.addWidget(self.btn_clean)
        self.log = QTextEdit()
        self.log.setReadOnly(True)
        layout.addWidget(self.log)

        self.setLayout(layout)

    def add_custom_path(self):
        p = self.custom_path_input.text().strip()
        if p:
            path = Path(p).expanduser()
            self.custom_paths.append(path)
            self.log.append(f"Добавлен пользовательский путь: {path}")
            self.custom_path_input.clear()

    def run_clean(self):
        self.log.clear()
        # Count tasks
        tasks = [opt for opt in self.options if self.checkboxes[opt].isChecked()]
        total = len(tasks) + len(self.custom_paths)
        self.progress.setMaximum(total)
        count = 0

        # Execute
        for label in tasks:
            try:
                self.options[label]()
                self.log.append(f"Выполнено: {label}")
            except Exception as e:
                self.log.append(f"Ошибка при {label}: {e}")
            count += 1
            self.progress.setValue(count)

        # Custom paths
        for path in self.custom_paths:
            try:
                self._delete_path(path)
                self.log.append(f"Удалено (пользовательский): {path}")
            except Exception as e:
                self.log.append(f"Ошибка удаления {path}: {e}")
            count += 1
            self.progress.setValue(count)

        # Scheduler
        if self.cb_schedule.isChecked():
            self.schedule_task()

        # Notifications
        subj = "Cleaner: Завершено"
        body = f"Очистка выполнена. Лог:\n" + self.log.toPlainText()
        if self.input_email.text().strip():
            self.send_email(self.input_email.text().strip(), subj, body)
        if self.input_slack.text().strip():
            self.send_slack(self.input_slack.text().strip(), body)

        QMessageBox.information(self, "Готово", "Все задачи завершены!")

    def _delete_path(self, path: Path):
        if path.is_dir(): shutil.rmtree(path, ignore_errors=True)
        else: path.unlink(missing_ok=True)

    def clean_temp(self):
        for d in [Path(os.getenv('TMP','/tmp')), Path(os.getenv('TEMPDIR','/tmp'))]:
            if d.exists():
                for i in d.iterdir(): self._delete_path(i)

    def clean_browser_cache(self):
        home = Path.home()
        for b in ["google-chrome","chromium","mozilla/firefox"]:
            for c in (home/".cache"/b).glob("**/Cache*"):
                self._delete_path(c)

    def clean_trash(self):
        t = Path.home()/".local/share/Trash/files"
        if t.exists():
            for i in t.iterdir(): self._delete_path(i)

    def clean_system_logs(self): subprocess.run(["journalctl","--vacuum-time=7d"])
    def clean_apt_cache(self):
        for p in [Path("/var/cache/apt/archives"),Path.home()/".cache/apt"]:
            if p.exists():
                for i in p.iterdir(): self._delete_path(i)
    def clean_pip_cache(self):
        p=Path.home()/".cache/pip";
        [self._delete_path(i) for i in p.iterdir()] if p.exists() else None
    def clean_thumbnails(self):
        p=Path.home()/".cache/thumbnails";
        [self._delete_path(i) for i in p.iterdir()] if p.exists() else None
    def clean_docker(self): subprocess.run(["docker","system","prune","-af"])
    def clean_flatpak_cache(self):
        for f in Path.home()/".var/app".glob("**/cache"): self._delete_path(f)
    def clean_snap_cache(self):
        p=Path.home()/".cache/snapd";
        [self._delete_path(i) for i in p.iterdir()] if p.exists() else None
    def clean_autoremove(self):
        if platform.system()=="Linux": subprocess.run(["sudo","apt","autoremove","-y"])
    
    def clean_defrag_all(self):
        system=platform.system()
        if system=="Windows":
            subprocess.run(["defrag","C:","/U","/V"])
        elif system=="Linux":
            subprocess.run(["sudo","e4defrag","/"], check=False)
            subprocess.run(["sudo","xfs_fsr","-v","/"], check=False)
            subprocess.run(["sudo","btrfs","filesystem","defragment","-r","/"], check=False)
        else:
            self.log.append(f"Дефрагментация не поддерживается на {system}")

    def clean_registry(self):
        if platform.system()=="Windows":
            subprocess.run(["reg","delete",
                r"HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU","/f"],check=False)
        else:
            self.log.append("Чистка реестра доступна только в Windows")

    def schedule_task(self):
        path=Path(sys.argv[0]).resolve();freq=self.schedule_combo.currentText()
        if platform.system()=="Linux":
            tmpl={"Ежедневно":"0 2 * * *","Еженедельно":"0 2 * * 0","Ежемесячно":"0 2 1 * *"}[freq]
            subprocess.run(f"(crontab -l; echo '{tmpl} python3 {path}') | crontab -", shell=True)
        elif platform.system()=="Windows":
            sch={"Ежедневно":"DAILY","Еженедельно":"WEEKLY","Ежемесячно":"MONTHLY"}[freq]
            subprocess.run(["schtasks","/Create","/SC",sch,"/TN","CleanerTask", 
                "/TR",f'"{sys.executable} {path}"',"/ST","02:00"],check=False)

    def send_email(self,to,subject,body):
        server=os.getenv('SMTP_SERVER');port=int(os.getenv('SMTP_PORT',587))
        user=os.getenv('EMAIL_USER');pwd=os.getenv('EMAIL_PASS')
        if not all([server,user,pwd]): return self.log.append("SMTP vars не настроены")
        msg=MIMEText(body)
        msg['Subject']=subject;msg['From']=user;msg['To']=to
        try:
            with smtplib.SMTP(server,port) as s:
                s.starttls();s.login(user,pwd);s.sendmail(user,[to],msg.as_string())
            self.log.append(f"Email отправлен: {to}")
        except Exception as e:
            self.log.append(f"Ошибка отправки Email: {e}")

    def send_slack(self,webhook,body):
        try:
            payload=json.dumps({"text":body})
            subprocess.run(["curl","-X","POST","-H","Content-Type: application/json",
                "-d",payload,webhook],check=False)
            self.log.append("Slack уведомление отправлено")
        except Exception as e:
            self.log.append(f"Ошибка Slack: {e}")

if __name__ == "__main__":
    app=QApplication(sys.argv)
    win=CleanerApp()
    win.show()
    sys.exit(app.exec_())
