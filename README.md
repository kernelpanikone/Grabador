import mss
import numpy as np
import threading
import time
import tkinter as tk
from tkinter import filedialog, messagebox


class GrabadorApp:
    # --- Paleta modo oscuro ---
    BG = "#1E1E1E"
    BG_PANEL = "#2A2A2A"
    TEXTO = "#E8E8E8"
    TEXTO_SUAVE = "#9A9A9A"
    VERDE = "#2ecc71"
    VERDE_HOVER = "#27ae60"
    ROJO = "#e74c3c"
    ROJO_HOVER = "#c0392b"
    GRIS_BOTON = "#3A3A3A"
    GRIS_BOTON_HOVER = "#4A4A4A"

    def __init__(self, root):
        self.root = root
        self.root.title("Grabador de Pantalla")
        self.root.geometry("380x220")
        self.root.resizable(False, False)
        self.root.configure(bg=self.BG)

        self.grabando = False
        self.hilo = None
        self.out = None
        self.archivo_salida = "grabacion.mp4"
        self.fps = 20.0
        self.inicio_tiempo = None

        # --- Interfaz ---
        self.lbl_estado = tk.Label(
            root, text="Listo para grabar", font=("Segoe UI", 12),
            bg=self.BG, fg=self.TEXTO
        )
        self.lbl_estado.pack(pady=(16, 4))

        self.lbl_archivo = tk.Label(
            root, text=f"Archivo: {self.archivo_salida}", font=("Segoe UI", 9),
            bg=self.BG, fg=self.TEXTO_SUAVE
        )
        self.lbl_archivo.pack()

        self.lbl_tiempo = tk.Label(
            root, text="", font=("Segoe UI", 14, "bold"),
            bg=self.BG, fg=self.TEXTO
        )
        self.lbl_tiempo.pack(pady=10)

        frame_botones = tk.Frame(root, bg=self.BG)
        frame_botones.pack(pady=10)

        self.btn_guardar_como = tk.Button(
            frame_botones, text="Guardar como...", width=13, command=self.elegir_archivo,
            bg=self.GRIS_BOTON, fg=self.TEXTO, activebackground=self.GRIS_BOTON_HOVER,
            activeforeground=self.TEXTO, relief="flat", bd=0, highlightthickness=0,
            cursor="hand2"
        )
        self.btn_guardar_como.grid(row=0, column=0, padx=5)

        self.btn_grabar = tk.Button(
            frame_botones, text="🔴 Grabar", width=13, command=self.iniciar_grabacion,
            bg=self.VERDE, fg="white", activebackground=self.VERDE_HOVER,
            activeforeground="white", relief="flat", bd=0, highlightthickness=0,
            cursor="hand2"
        )
        self.btn_grabar.grid(row=0, column=1, padx=5)

        self.btn_detener = tk.Button(
            frame_botones, text="⏹ Detener", width=13, command=self.detener_grabacion,
            state="disabled", bg=self.ROJO, fg="white", activebackground=self.ROJO_HOVER,
            activeforeground="white", relief="flat", bd=0, highlightthickness=0,
            disabledforeground="#888888", cursor="hand2"
        )
        self.btn_detener.grid(row=0, column=2, padx=5)

        self.root.protocol("WM_DELETE_WINDOW", self.al_cerrar)

    # --- Elegir dónde guardar el video (antes de grabar) ---
    def elegir_archivo(self):
        ruta = filedialog.asksaveasfilename(
            defaultextension=".mp4",
            filetypes=[("Video MP4", "*.mp4")],
            initialfile="grabacion.mp4",
            title="Guardar grabación como...",
        )
        if ruta:
            self.archivo_salida = ruta
            self.lbl_archivo.config(text=f"Archivo: {ruta}")

    # --- Iniciar grabación ---
    def iniciar_grabacion(self):
        if self.grabando:
            return
        self.grabando = True
        self.inicio_tiempo = time.time()

        self.btn_grabar.config(state="disabled")
        self.btn_detener.config(state="normal")
        self.btn_guardar_como.config(state="disabled")
        self.lbl_estado.config(text="🔴 Grabando...")

        self.hilo = threading.Thread(target=self._grabar_en_segundo_plano, daemon=True)
        self.hilo.start()
        self._actualizar_tiempo()

    # --- Loop de captura, corre en un hilo aparte ---
    def _grabar_en_segundo_plano(self):
        try:
            with mss.mss() as sct:
                monitor = sct.monitors[1]
                width = monitor["width"]
                height = monitor["height"]

                fourcc = cv2.VideoWriter_fourcc(*"mp4v")
                self.out = cv2.VideoWriter(self.archivo_salida, fourcc, self.fps, (width, height))

                while self.grabando:
                    img = np.array(sct.grab(monitor))
                    frame = cv2.cvtColor(img, cv2.COLOR_BGRA2BGR)
                    self.out.write(frame)
        except Exception as e:
            self.grabando = False
            # messagebox debe llamarse desde el hilo principal
            self.root.after(0, lambda: messagebox.showerror("Error al grabar", str(e)))
        finally:
            if self.out is not None:
                self.out.release()
                self.out = None

    # --- Detener grabación y guardar ---
    def detener_grabacion(self):
        if not self.grabando:
            return
        self.grabando = False
        if self.hilo is not None:
            self.hilo.join(timeout=5)

        self.btn_grabar.config(state="normal")
        self.btn_detener.config(state="disabled")
        self.btn_guardar_como.config(state="normal")
        self.lbl_estado.config(text="✅ Grabación guardada")

        messagebox.showinfo("Grabación guardada", f"El video se guardó en:\n{self.archivo_salida}")

    # --- Reloj en pantalla mientras graba ---
    def _actualizar_tiempo(self):
        if self.grabando:
            segundos = int(time.time() - self.inicio_tiempo)
            minutos, segundos = divmod(segundos, 60)
            self.lbl_tiempo.config(text=f"{minutos:02d}:{segundos:02d}")
            self.root.after(500, self._actualizar_tiempo)
        else:
            self.lbl_tiempo.config(text="")

    # --- Si cierran la ventana mientras graba, preguntar ---
    def al_cerrar(self):
        if self.grabando:
            if messagebox.askyesno("Grabación en curso", "Hay una grabación activa. ¿Detener y salir?"):
                self.detener_grabacion()
                self.root.destroy()
        else:
            self.root.destroy()


if __name__ == "__main__":
    root = tk.Tk()
    app = GrabadorApp(root)
    root.mainloop()
-invention
