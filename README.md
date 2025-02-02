import tkinter as tk
from tkinter import ttk, messagebox
import math


class BeamApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Расчет балки на прочность")
        self.root.geometry("1400x900")

        # Стили
        self.style = ttk.Style()
        self.style.configure("TFrame", background="#f0f0f0")
        self.style.configure("TLabel", background="#f0f0f0", font=('Arial', 10))
        self.style.configure("Header.TLabel", font=('Arial', 12, 'bold'), foreground="#2c3e50")

        # Главный контейнер
        self.main_frame = ttk.Frame(root)
        self.main_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)

        # Левая панель (ввод данных)
        self.left_panel = ttk.Frame(self.main_frame, width=450)
        self.left_panel.pack(side=tk.LEFT, fill=tk.Y, padx=10, pady=10)

        # Правая панель (визуализация)
        self.right_panel = ttk.Frame(self.main_frame)
        self.right_panel.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=10, pady=10)

        # === Левая панель ===
        # Секция: Параметры балки
        beam_frame = ttk.LabelFrame(self.left_panel, text="📏 Параметры балки", padding=15)
        beam_frame.pack(fill=tk.X, pady=10)
        self.length_entry = self.create_input_field(beam_frame, "Длина балки (м):", "5.0")
        self.width_entry = self.create_input_field(beam_frame, "Ширина сечения (м):", "0.2")
        self.height_entry = self.create_input_field(beam_frame, "Высота сечения (м):", "0.4")

        # Секция: Характеристики материала
        material_frame = ttk.LabelFrame(self.left_panel, text="🧱 Характеристики материала", padding=15)
        material_frame.pack(fill=tk.X, pady=10)
        self.E_entry = self.create_input_field(material_frame, "Модуль упругости (Па):", "2.1e11")
        self.strength_entry = self.create_input_field(material_frame, "Предел прочности (Па):", "250e6")

        # Секция: Опоры (вкладки)
        self.supports_notebook = ttk.Notebook(self.left_panel)
        self.supports_notebook.pack(fill=tk.X, pady=10)

        # Вкладка: Подвижная/неподвижная опора
        simple_support_tab = ttk.Frame(self.supports_notebook)
        self.supports_notebook.add(simple_support_tab, text="📌 Опора")
        self.fixed_support_entry = self.create_input_field(simple_support_tab, "Смещение неподвижной (м):", "1.0")
        self.movable_support_entry = self.create_input_field(simple_support_tab, "Смещение подвижной (м):", "4.0")

        # Вкладка: Жесткая заделка
        fixed_support_tab = ttk.Frame(self.supports_notebook)
        self.supports_notebook.add(fixed_support_tab, text="🔒 Заделка")
        self.rigid_support_entry = self.create_input_field(fixed_support_tab, "Смещение заделки (м):", "0.0")
        self.rigid_angle_entry = self.create_input_field(fixed_support_tab, "Угол поворота (°):", "0")

        # Секция: Нагрузки
        self.forces = []  # Список для хранения сил
        self.moments = []  # Список для хранения моментов
        self.distributed_loads = []  # Список для хранения распределенных нагрузок

        # Фрейм для управления нагрузками
        loads_frame = ttk.LabelFrame(self.left_panel, text="📦 Нагрузки", padding=15)
        loads_frame.pack(fill=tk.X, pady=10)

        # Кнопки для добавления нагрузок
        ttk.Button(loads_frame, text="➕ Добавить силу", command=self.add_force).pack(fill=tk.X, pady=5)
        ttk.Button(loads_frame, text="➕ Добавить момент", command=self.add_moment).pack(fill=tk.X, pady=5)
        ttk.Button(loads_frame, text="➕ Добавить распределенную нагрузку", command=self.add_distributed_load).pack(fill=tk.X, pady=5)

        # Кнопка расчета
        self.calculate_btn = ttk.Button(self.left_panel, text="🔄 Рассчитать", command=self.calculate)
        self.calculate_btn.pack(fill=tk.X, pady=15)

        # === Правая панель ===
        # Визуализация балки
        self.canvas = tk.Canvas(self.right_panel, bg="white", bd=2, relief=tk.GROOVE, height=500)
        self.canvas.pack(fill=tk.BOTH, expand=True, pady=10)

        # Информационная панель
        self.info_text = tk.Text(self.right_panel, height=10, wrap=tk.WORD, font=('Courier', 10))
        self.info_text.pack(fill=tk.X, pady=10)
        self.update_info()

        # Привязка событий
        self.bind_events()

    def create_input_field(self, parent, label, default):
        frame = ttk.Frame(parent)
        frame.pack(fill=tk.X, pady=5)

        lbl = ttk.Label(frame, text=label, width=25, anchor=tk.W)
        lbl.pack(side=tk.LEFT)

        entry = ttk.Entry(frame)
        entry.insert(0, default)
        entry.pack(side=tk.RIGHT, fill=tk.X, expand=True)
        return entry

    def add_force(self):
        """Добавляет новую силу."""
        force_frame = ttk.Frame(self.left_panel)
        force_frame.pack(fill=tk.X, pady=5)

        # Поля для ввода
        force_entry = self.create_input_field(force_frame, f"Сила F{len(self.forces) + 1} (Н):", "0")
        force_pos_entry = self.create_input_field(force_frame, f"Позиция F{len(self.forces) + 1} (м):", "0")
        force_dir = ttk.Combobox(force_frame, values=["↓ Вниз", "↑ Вверх"], state="readonly")
        force_dir.set("↓ Вниз")
        force_dir.pack(side=tk.RIGHT, fill=tk.X, expand=True)

        # Кнопка удаления
        remove_btn = ttk.Button(force_frame, text="❌", width=3, command=lambda: self.remove_load(force_frame, self.forces))
        remove_btn.pack(side=tk.RIGHT, padx=5)

        # Добавляем в список
        self.forces.append((force_entry, force_pos_entry, force_dir, force_frame))
        self.update_display()

    def add_moment(self):
        """Добавляет новый момент."""
        moment_frame = ttk.Frame(self.left_panel)
        moment_frame.pack(fill=tk.X, pady=5)

        # Поля для ввода
        moment_entry = self.create_input_field(moment_frame, f"Момент M{len(self.moments) + 1} (Н·м):", "0")
        moment_pos_entry = self.create_input_field(moment_frame, f"Позиция M{len(self.moments) + 1} (м):", "0")
        moment_dir = ttk.Combobox(moment_frame, values=["↻ По часовой", "↺ Против часовой"], state="readonly")
        moment_dir.set("↻ По часовой")
        moment_dir.pack(side=tk.RIGHT, fill=tk.X, expand=True)

        # Кнопка удаления
        remove_btn = ttk.Button(moment_frame, text="❌", width=3, command=lambda: self.remove_load(moment_frame, self.moments))
        remove_btn.pack(side=tk.RIGHT, padx=5)

        # Добавляем в список
        self.moments.append((moment_entry, moment_pos_entry, moment_dir, moment_frame))
        self.update_display()

    def add_distributed_load(self):
        """Добавляет новую распределенную нагрузку."""
        dist_frame = ttk.Frame(self.left_panel)
        dist_frame.pack(fill=tk.X, pady=5)

        # Поля для ввода
        dist_entry = self.create_input_field(dist_frame, f"Нагрузка q{len(self.distributed_loads) + 1} (Н/м):", "0")
        dist_start_entry = self.create_input_field(dist_frame, f"Начало q{len(self.distributed_loads) + 1} (м):", "0")
        dist_end_entry = self.create_input_field(dist_frame, f"Конец q{len(self.distributed_loads) + 1} (м):", "0")

        # Кнопка удаления
        remove_btn = ttk.Button(dist_frame, text="❌", width=3, command=lambda: self.remove_load(dist_frame, self.distributed_loads))
        remove_btn.pack(side=tk.RIGHT, padx=5)

        # Добавляем в список
        self.distributed_loads.append((dist_entry, dist_start_entry, dist_end_entry, dist_frame))
        self.update_display()

    def remove_load(self, frame, load_list):
        """Удаляет нагрузку."""
        frame.destroy()
        load_list.remove(next(item for item in load_list if item[-1] == frame))
        self.update_display()

    def bind_events(self):
        entries = [
            self.length_entry, self.width_entry, self.height_entry,
            self.E_entry, self.strength_entry, self.fixed_support_entry,
            self.movable_support_entry, self.rigid_support_entry,
            self.rigid_angle_entry
        ]
        for entry in entries:
            entry.bind("<KeyRelease>", lambda e: self.update_display())

    def update_display(self, event=None):
        try:
            self.update_beam()
            self.update_info()
        except Exception as e:
            print(f"Ошибка обновления: {str(e)}")

    def update_beam(self):
        self.canvas.delete("all")

        # Фиксированная длина балки в пикселях
        fixed_beam_length_px = 800

        # Получаем длину балки из входных данных
        length_m = float(self.length_entry.get())

        # Масштаб для перевода метров в пиксели
        scale = fixed_beam_length_px / length_m

        # Параметры отрисовки
        canvas_width = self.canvas.winfo_width()
        canvas_height = self.canvas.winfo_height()

        # Центрируем балку на холсте
        beam_start = (canvas_width - fixed_beam_length_px) // 2
        beam_end = beam_start + fixed_beam_length_px
        beam_y = canvas_height // 2

        # Отрисовка балки
        self.canvas.create_line(beam_start, beam_y, beam_end, beam_y, width=4, fill="#3498db", tag="beam")

        # Отображение длины балки
        self.canvas.create_text(
            (beam_start + beam_end) // 2, beam_y + 30,
            text=f"Длина балки: {length_m} м",
            fill="#2c3e50", font=('Arial', 12, 'bold')
        )

        # Отрисовка опор
        self.draw_supports(scale, beam_start, beam_y)

        # Отрисовка нагрузок
        self.draw_forces(scale, beam_start, beam_y)
        self.draw_moments(scale, beam_start, beam_y)
        self.draw_distributed_loads(scale, beam_start, beam_y)

    def draw_supports(self, scale, offset, beam_y):
        # Неподвижная опора
        fixed_pos = float(self.fixed_support_entry.get())
        if fixed_pos > 0:
            x = offset + fixed_pos * scale
            self.canvas.create_polygon(
                x - 15, beam_y + 20,
                x + 15, beam_y + 20,
                x, beam_y,
                fill="#2c3e50", outline="black"
            )

        # Подвижная опора
        movable_pos = float(self.movable_support_entry.get())
        if movable_pos > 0:
            x = offset + movable_pos * scale
            self.canvas.create_oval(
                x - 12, beam_y + 8,
                x + 12, beam_y + 20,
                fill="#2c3e50", outline="black"
            )

        # Жесткая заделка
        rigid_pos = float(self.rigid_support_entry.get())
        if rigid_pos > 0:
            x = offset + rigid_pos * scale
            angle = math.radians(float(self.rigid_angle_entry.get()))
            self.canvas.create_polygon(
                x - 15, beam_y + 20,
                x + 15, beam_y + 20,
                x, beam_y,
                fill="#8e44ad", outline="black"
            )
            # Отрисовка угла поворота
            arrow_len = 30
            end_x = x + arrow_len * math.cos(angle)
            end_y = beam_y - arrow_len * math.sin(angle)
            self.canvas.create_line(
                x, beam_y,
                end_x, end_y,
                width=2, fill="#8e44ad", arrow=tk.LAST
            )

    def draw_forces(self, scale, offset, beam_y):
        for i, (force_entry, pos_entry, direction, _) in enumerate(self.forces):
            force = float(force_entry.get())
            if force == 0:
                continue

            pos = offset + float(pos_entry.get()) * scale
            direction = -1 if "↓" in direction.get() else 1
            arrow_size = 40

            self.canvas.create_line(
                pos, beam_y,
                pos, beam_y + direction * arrow_size,
                width=3, fill="#e74c3c", arrow=tk.LAST
            )
            self.canvas.create_text(
                pos + 15, beam_y + direction * (arrow_size + 10),
                text=f"F{i + 1} = {force} Н",
                anchor=tk.W, fill="#e74c3c", font=('Arial', 10, 'bold')
            )

    def draw_moments(self, scale, offset, beam_y):
        for i, (moment_entry, pos_entry, direction, _) in enumerate(self.moments):
            moment = float(moment_entry.get())
            if moment == 0:
                continue

            pos = offset + float(pos_entry.get()) * scale
            radius = 20
            self.canvas.create_oval(
                pos - radius, beam_y - radius,
                pos + radius, beam_y + radius,
                outline="#8e44ad", width=3
            )
            # Отрисовка направления
            angle = 45 if "↻" in direction.get() else -45
            self.canvas.create_arc(
                pos - radius, beam_y - radius,
                pos + radius, beam_y + radius,
                start=angle, extent=180,
                style=tk.ARC, outline="#8e44ad", width=3
            )
            self.canvas.create_text(
                pos + radius + 10, beam_y,
                text=f"M{i + 1} = {moment} Н·м",
                anchor=tk.W, fill="#8e44ad", font=('Arial', 10, 'bold')
            )

    def draw_distributed_loads(self, scale, offset, beam_y):
        for i, (dist_entry, start_entry, end_entry, _) in enumerate(self.distributed_loads):
            q = float(dist_entry.get())
            if q == 0:
                continue

            start = offset + float(start_entry.get()) * scale
            end = offset + float(end_entry.get()) * scale

            # Отрисовка распределенной нагрузки
            step = 20  # Шаг между стрелками
            num_arrows = int((end - start) // step)
            for x in [start + i * step for i in range(num_arrows + 1)]:
                if x > end:
                    continue
                self.canvas.create_line(
                    x, beam_y,
                    x, beam_y + 30,
                    width=1, fill="#2ecc71", arrow=tk.LAST
                )
            self.canvas.create_text(
                (start + end) // 2, beam_y + 40,
                text=f"q{i + 1} = {q} Н/м",
                fill="#2ecc71", font=('Arial', 10, 'bold')
            )

    def update_info(self):
        text = f"""📊 Текущие параметры:
Длина балки: {self.length_entry.get()} м
Сечение: {self.width_entry.get()} м × {self.height_entry.get()} м
Модуль упругости: {self.E_entry.get()} Па
Предел прочности: {self.strength_entry.get()} Па
Неподвижная опора: {self.fixed_support_entry.get()} м
Подвижная опора: {self.movable_support_entry.get()} м
Жесткая заделка: {self.rigid_support_entry.get()} м (угол {self.rigid_angle_entry.get()}°)
"""
        # Добавляем информацию о нагрузках
        for i, (force_entry, pos_entry, direction, _) in enumerate(self.forces):
            text += f"Сила F{i + 1}: {force_entry.get()} Н на {pos_entry.get()} м ({direction.get()})\n"
        for i, (moment_entry, pos_entry, direction, _) in enumerate(self.moments):
            text += f"Момент M{i + 1}: {moment_entry.get()} Н·м на {pos_entry.get()} м ({direction.get()})\n"
        for i, (dist_entry, start_entry, end_entry, _) in enumerate(self.distributed_loads):
            text += f"Распределенная нагрузка q{i + 1}: {dist_entry.get()} Н/м от {start_entry.get()} м до {end_entry.get()} м\n"

        self.info_text.config(state=tk.NORMAL)
        self.info_text.delete(1.0, tk.END)
        self.info_text.insert(tk.END, text)
        self.info_text.config(state=tk.DISABLED)

    def calculate(self):
        try:
            # Здесь будет код расчетов
            messagebox.showinfo("Расчет", "Расчеты успешно выполнены!")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Некорректные данные: {str(e)}")


if __name__ == "__main__":
    root = tk.Tk()
    app = BeamApp(root)
    root.mainloop()
