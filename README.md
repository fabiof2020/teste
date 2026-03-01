from kivy.lang import Builder
from kivymd.app import MDApp
from kivy.uix.screenmanager import ScreenManager, Screen
from kivymd.uix.list import OneLineListItem
from kivymd.uix.dialog import MDDialog
from kivymd.uix.button import MDFlatButton
import sqlite3
import hashlib

KV = '''
ScreenManager:
    LoginScreen:
    MainScreen:

<LoginScreen>:
    name: "login"

    MDBoxLayout:
        orientation: "vertical"
        padding: 40
        spacing: 20

        MDLabel:
            text: "Seja Bem Vindo"
            halign: "center"
            font_style: "H2"

        MDTextField:
            id: usuario
            hint_text: "Usuário"

        MDTextField:
            id: senha
            hint_text: "Senha"
            password: True

        MDRaisedButton:
            text: "Entrar"
            on_release: app.fazer_login(usuario.text, senha.text)

<MainScreen>:
    name: "main"

    MDBoxLayout:
        orientation: "vertical"

        MDTopAppBar:
            title: "Cadastro de Câmeras"

        MDTextField:
            id: busca
            hint_text: "Buscar"
            on_text: app.atualizar(self.text)
            size_hint_x: 0.95
            pos_hint: {"center_x": 0.5}

        ScrollView:
            MDList:
                id: lista

        MDBoxLayout:
            size_hint_y: None
            height: "60dp"
            spacing: 10
            padding: 10

            MDRaisedButton:
                text: "Adicionar"
                on_release: app.abrir_dialog()

            MDRaisedButton:
                text: "Exportar Excel"
                on_release: app.exportar_excel()
'''

class LoginScreen(Screen):
    pass

class MainScreen(Screen):
    pass

class App(MDApp):

    def build(self):
        self.theme_cls.primary_palette = "Blue"
        self.criar_banco()
        return Builder.load_string(KV)

    def conectar(self):
        return sqlite3.connect("cameras.db")

    def criar_banco(self):
        conn = self.conectar()
        cursor = conn.cursor()

        cursor.execute("""
        CREATE TABLE IF NOT EXISTS usuarios (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            usuario TEXT,
            senha TEXT
        )
        """)

        cursor.execute("""
        CREATE TABLE IF NOT EXISTS cameras (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            numero TEXT,
            ip TEXT
        )
        """)

        cursor.execute("SELECT * FROM usuarios WHERE usuario=?", ("admin",))
        if not cursor.fetchone():
            senha_hash = hashlib.sha256("1234".encode()).hexdigest()
            cursor.execute("INSERT INTO usuarios (usuario, senha) VALUES (?, ?)",
                           ("admin", senha_hash))

        conn.commit()
        conn.close()

    def fazer_login(self, usuario, senha):
        conn = self.conectar()
        cursor = conn.cursor()

        senha_hash = hashlib.sha256(senha.encode()).hexdigest()
        cursor.execute("SELECT * FROM usuarios WHERE usuario=? AND senha=?",
                       (usuario, senha_hash))

        user = cursor.fetchone()
        conn.close()

        if user:
            self.root.current = "main"
            self.atualizar()
        else:
            self.alerta("Usuário ou senha incorretos")

    def alerta(self, msg):
        dialog = MDDialog(
            text=msg,
            buttons=[MDFlatButton(text="OK", on_release=lambda x: dialog.dismiss())],
        )
        dialog.open()

    # ------------------------
    # CRUD
    # ------------------------

    def abrir_dialog(self, cam_id=None, numero="", ip=""):
        self.cam_editando = cam_id

        self.dialog = MDDialog(
            title="Câmera",
            type="custom",
            content_cls=Builder.load_string(f'''
MDBoxLayout:
    orientation: "vertical"
    spacing: 10
    size_hint_y: None
    height: "150dp"

    MDTextField:
        id: numero
        hint_text: "Número"
        text: "{numero}"

    MDTextField:
        id: ip
        hint_text: "IP"
        text: "{ip}"
'''),
            buttons=[
                MDFlatButton(text="Cancelar",
                             on_release=lambda x: self.dialog.dismiss()),
                MDFlatButton(text="Salvar",
                             on_release=lambda x: self.salvar_camera())
            ],
        )
        self.dialog.open()

    def salvar_camera(self):
        numero = self.dialog.content_cls.ids.numero.text
        ip = self.dialog.content_cls.ids.ip.text

        if not numero or not ip:
            self.alerta("Preencha todos os campos")
            return

        conn = self.conectar()
        cursor = conn.cursor()

        if self.cam_editando:
            cursor.execute("UPDATE cameras SET numero=?, ip=? WHERE id=?",
                           (numero, ip, self.cam_editando))
        else:
            cursor.execute("INSERT INTO cameras (numero, ip) VALUES (?, ?)",
                           (numero, ip))

        conn.commit()
        conn.close()

        self.dialog.dismiss()
        self.atualizar()

    def confirmar_exclusao(self, cam_id):
        dialog = MDDialog(
            text="Deseja realmente excluir?",
            buttons=[
                MDFlatButton(text="Não",
                             on_release=lambda x: dialog.dismiss()),
                MDFlatButton(text="Sim",
                             on_release=lambda x: self.excluir(cam_id, dialog))
            ],
        )
        dialog.open()

    def excluir(self, cam_id, dialog):
        conn = self.conectar()
        cursor = conn.cursor()
        cursor.execute("DELETE FROM cameras WHERE id=?", (cam_id,))
        conn.commit()
        conn.close()

        dialog.dismiss()
        self.atualizar()

    def atualizar(self, texto=""):
        lista = self.root.get_screen("main").ids.lista
        lista.clear_widgets()

        conn = self.conectar()
        cursor = conn.cursor()

        cursor.execute("""
        SELECT * FROM cameras
        WHERE numero LIKE ? OR ip LIKE ?
        """, (f"%{texto}%", f"%{texto}%"))

        dados = cursor.fetchall()
        conn.close()

        for cam in dados:
            item = OneLineListItem(
                text=f"{cam[1]} - {cam[2]}",
                on_release=lambda x, c=cam: self.menu_item(c)
            )
            lista.add_widget(item)

    def menu_item(self, cam):
        dialog = MDDialog(
            text="O que deseja fazer?",
            buttons=[
                MDFlatButton(text="Editar",
                             on_release=lambda x: [dialog.dismiss(),
                                                   self.abrir_dialog(cam[0], cam[1], cam[2])]),
                MDFlatButton(text="Excluir",
                             on_release=lambda x: [dialog.dismiss(),
                                                   self.confirmar_exclusao(cam[0])]),
                MDFlatButton(text="Cancelar",
                             on_release=lambda x: dialog.dismiss()),
            ],
        )
        dialog.open()
        
    def exportar_excel(self):
        from openpyxl import Workbook
        import os

        conn = self.conectar()
        cursor = conn.cursor()
        cursor.execute("SELECT numero, ip FROM cameras")
        dados = cursor.fetchall()
        conn.close()

        if not dados:
            self.alerta("Nenhuma câmera cadastrada")
            return

        wb = Workbook()
        ws = wb.active
        ws.title = "Cameras"

        ws.append(["Número", "IP"])

        for linha in dados:
            ws.append(linha)

        caminho = "/storage/emulated/0/cameras.xlsx"
        wb.save(caminho)

        self.alerta(f"Arquivo salvo em:\n{caminho}")


App().run()
