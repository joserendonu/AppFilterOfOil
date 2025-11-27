# AppFilterOfOil
AppFilterOfOil

#CODIGO DE PYEDITER ESE
VOY A INTENTAR CON LAS IMAGENES LLENAR EL INVENTARIO DE TAL MANERA QUE SOLO SEA BUSCAR



import os
import json
import subprocess
from kivy.app import App
from kivy.metrics import dp
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.image import Image
from kivy.uix.textinput import TextInput
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.uix.popup import Popup
from kivy.uix.scrollview import ScrollView
from kivy.uix.filechooser import FileChooserListView
from datetime import datetime

DATA_FILE = "inventario.json"
IMG_FOLDER = "fotos"

if not os.path.exists(IMG_FOLDER):
    os.makedirs(IMG_FOLDER)


class InventarioApp(App):

    def build(self):
        self.items = self.cargar_inventario()

        root = BoxLayout(orientation="vertical")

        btn_add = Button(
            text="➕ Agregar producto",
            size_hint=(1, None),
            height=dp(60),
            background_color=(0.2, 0.6, 1, 1)
        )
        btn_add.bind(on_press=self.popup_agregar)

        root.add_widget(btn_add)

        scroll = ScrollView(size_hint=(1, 1))
        self.lista = GridLayout(cols=1, spacing=dp(8), size_hint_y=None, padding=dp(10))
        self.lista.bind(minimum_height=self.lista.setter("height"))
        scroll.add_widget(self.lista)
        root.add_widget(scroll)

        self.actualizar_lista()
        return root

    # -----------------------------------
    # CARGAR Y GUARDAR INVENTARIO
    # -----------------------------------
    def cargar_inventario(self):
        if os.path.exists(DATA_FILE):
            with open(DATA_FILE, "r") as f:
                return json.load(f)
        return []

    def guardar_inventario(self):
        with open(DATA_FILE, "w") as f:
            json.dump(self.items, f)

    # -----------------------------------
    # POPUP PARA AGREGAR PRODUCTO
    # -----------------------------------
    def popup_agregar(self, *args):
        layout = BoxLayout(orientation="vertical", padding=dp(10), spacing=dp(10))

        self.in_nombre = TextInput(hint_text="Nombre", size_hint_y=None, height=dp(45))
        self.in_desc = TextInput(hint_text="Descripción", size_hint_y=None, height=dp(100))

        btn_foto = Button(text="📸 Tomar foto (cámara nativa)", size_hint_y=None, height=dp(45))
        btn_foto.bind(on_press=self.tomar_foto)

        btn_galeria = Button(text="🖼 Seleccionar foto de galería", size_hint_y=None, height=dp(45))
        btn_galeria.bind(on_press=self.abrir_galeria)

        btn_guardar = Button(text="Guardar", background_color=(0, 1, 0, 1), size_hint_y=None, height=dp(45))
        btn_guardar.bind(on_press=self.guardar_item)

        layout.add_widget(self.in_nombre)
        layout.add_widget(self.in_desc)
        layout.add_widget(btn_foto)
        layout.add_widget(btn_galeria)
        layout.add_widget(btn_guardar)

        self.ruta_foto = ""
        self.popup = Popup(title="Agregar producto", content=layout, size_hint=(0.9, 0.9))
        self.popup.open()

    # -----------------------------------
    # TOMAR FOTO SIN PLYER (CÁMARA NATIVA)
    # -----------------------------------
    def tomar_foto(self, *args):
        nombre = f"foto_{datetime.now().strftime('%Y%m%d_%H%M%S')}.jpg"
        ruta = os.path.join(IMG_FOLDER, nombre)

        # Intent nativo de Android
        cmd = [
            "am", "start",
            "-a", "android.media.action.IMAGE_CAPTURE",
            "-o", ruta
        ]

        try:
            subprocess.run(cmd)
            self.ruta_foto = ruta
            self.alerta("📸 La cámara se abrió.\nCuando tomes la foto, volverás aquí.")
        except Exception as e:
            self.alerta("❌ Error al abrir la cámara.\nActiva permisos para Pydroid.")

    # -----------------------------------
    # SELECCIONAR FOTO DESDE GALERÍA
    # -----------------------------------
    def abrir_galeria(self, *args):
        chooser = FileChooserListView(filters=[".jpg", ".png", "*.jpeg"])

        btn_sel = Button(text="Seleccionar", size_hint_y=None, height=dp(50))

        layout = BoxLayout(orientation="vertical")
        layout.add_widget(chooser)
        layout.add_widget(btn_sel)

        pop = Popup(title="Seleccionar imagen", content=layout, size_hint=(0.9, 0.9))

        def seleccionar(instance):
            if chooser.selection:
                self.ruta_foto = chooser.selection[0]
                pop.dismiss()

        btn_sel.bind(on_press=seleccionar)
        pop.open()

    # -----------------------------------
    # GUARDAR PRODUCTO
    # -----------------------------------
    def guardar_item(self, *args):
        nombre = self.in_nombre.text.strip()
        desc = self.in_desc.text.strip()

        if nombre == "":
            self.alerta("⚠ El nombre no puede estar vacío.")
            return

        item = {
            "nombre": nombre,
            "desc": desc,
            "img": self.ruta_foto
        }

        self.items.append(item)
        self.guardar_inventario()
        self.actualizar_lista()
        self.popup.dismiss()

    # -----------------------------------
    # ACTUALIZAR LISTA
    # -----------------------------------
    def actualizar_lista(self):
        self.lista.clear_widgets()

        for item in self.items:
            fila = BoxLayout(size_hint_y=None, height=dp(140), padding=dp(5), spacing=dp(10))

            if item["img"] and os.path.exists(item["img"]):
                img = Image(source=item["img"], size_hint=(0.35, 1))
            else:
                img = Label(text="Sin foto", size_hint=(0.35, 1))

            info = BoxLayout(orientation="vertical", size_hint=(0.65, 1))
            info.add_widget(Label(text=f"[b]{item['nombre']}[/b]", markup=True))
            info.add_widget(Label(text=item["desc"]))

            fila.add_widget(img)
            fila.add_widget(info)

            self.lista.add_widget(fila)

    # -----------------------------------
    # ALERTA POPUP
    # -----------------------------------
    def alerta(self, mensaje):
        pop = Popup(title="Información",
                    content=Label(text=mensaje),
                    size_hint=(0.8, 0.3))
        pop.open()


if _name_ == "_main_":
    InventarioApp().run()
