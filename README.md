[ai_racer_4_simple.py](https://github.com/user-attachments/files/32665645/ai_racer_4_simple.py)
# ai-racer-4
ии учится гонкам
import tkinter as tk
import random
import math
import json
import os

# ============================================================
# AI RACER 4 - SIMPLE
# 4 ИИ-гонщика. Победитель становится/остаётся чемпионом.
# Только Python + tkinter (стандартная библиотека).
# ============================================================

WIDTH = 900
HEIGHT = 600

CX = WIDTH // 2
CY = HEIGHT // 2

RX = 330
RY = 210

CAR_W = 30
CAR_H = 16

LAPS = 3
DT = 0.02

COLORS = ["#4aa3ff", "#ff6378", "#55d68d", "#c58cff"]
WHITE = "#eeeeee"
GRAY = "#8893a2"
BG = "#101419"
ROAD = "#3a4049"
GREEN = "#102117"


def clamp(x, a, b):
    return max(a, min(b, x))


class Brain:
    """Набор чисел, который эволюционирует после гонок."""

    def __init__(self, genes=None):
        if genes is None:
            # [максимальная скорость, ускорение,
            #  эффективность в поворотах, осторожность]
            self.genes = [
                random.uniform(0.75, 1.0),
                random.uniform(0.70, 1.0),
                random.uniform(0.60, 1.0),
                random.uniform(0.50, 1.0),
            ]
        else:
            self.genes = list(genes)

    def mutate(self):
        new_genes = []

        for g in self.genes:
            if random.random() < 0.8:
                g += random.gauss(0, 0.06)
            new_genes.append(clamp(g, 0.2, 1.2))

        return Brain(new_genes)


class Car:
    def __init__(self, name, brain, color):
        self.name = name
        self.brain = brain
        self.color = color
        self.reset(0.0)

    def reset(self, start):
        self.pos = start
        self.last_pos = start
        self.lap = 0
        self.speed = 0.0
        self.total = 0.0
        self.finished = False
        self.wobble = random.uniform(-0.02, 0.02)

    def update(self):
        if self.finished:
            return

        # Главная скорость зависит от "генов" мозга.
        max_speed = (
            0.72
            + self.brain.genes[0] * 0.23
            + self.brain.genes[1] * 0.10
        )

        # Кривизна условной трассы.
        curve = 0.5 + 0.5 * abs(math.sin(self.pos * math.pi * 2))

        # Более осторожные ИИ теряют меньше скорости на поворотах.
        corner_loss = curve * (1.0 - self.brain.genes[2]) * 0.20

        target = max_speed - corner_loss

        # Разгоняемся плавно.
        self.speed += (target - self.speed) * 0.045

        # Небольшая индивидуальность.
        self.speed += self.wobble
        self.speed = clamp(self.speed, 0.25, 1.20)

        old = self.pos
        self.pos += self.speed * DT
        self.total += self.speed * DT

        # Прошли линию старта.
        if old < 0.98 and self.pos >= 1.0:
            self.pos -= 1.0
            self.lap += 1

        if self.lap >= LAPS:
            self.finished = True


class Tournament:
    def __init__(self):
        self.generation = 1
        self.race_number = 0
        self.streak = 0
        self.record = 0

        self.champion = Car(
            "CHAMPION-1",
            Brain(),
            COLORS[0]
        )

        self.cars = []
        self.time = 0.0
        self.new_race()

    def new_race(self):
        self.race_number += 1
        self.time = 0.0
        self.cars = [self.champion]

        # ТРИ соперника.
        for i in range(3):
            # Чаще всего соперники - дети чемпиона.
            if random.random() < 0.85:
                brain = self.champion.brain.mutate()
            else:
                brain = Brain()

            self.cars.append(
                Car(
                    "RIVAL-%d" % (i + 1),
                    brain,
                    COLORS[i + 1]
                )
            )

        # Стартовые позиции немного разнесены.
        for i, car in enumerate(self.cars):
            car.reset(i * 0.001)

    def step(self):
        self.time += DT

        for car in self.cars:
            car.update()

        # Первый финишировавший.
        finished = [c for c in self.cars if c.finished]

        if finished:
            # В этой простой модели тот, кто первым пересёк 3 круга,
            # попадает сюда первым по обновлению списка.
            winner = max(self.cars, key=lambda c: (c.lap, c.pos))
            return self.finish(winner)

        # Аварийный таймаут.
        if self.time > 25:
            winner = max(self.cars, key=lambda c: (c.lap, c.pos))
            return self.finish(winner)

        return None

    def finish(self, winner):
        old_champion = self.champion

        if winner is old_champion:
            self.streak += 1
        else:
            self.generation += 1
            self.streak = 1

            # Новый чемпион.
            winner.name = "CHAMPION-%d" % self.generation
            winner.brain = winner.brain.mutate()
            self.champion = winner

        self.record = max(self.record, self.streak)

        result = (winner.name, winner is old_champion)

        self.new_race()
        return result

    def save(self):
        data = {
            "genes": self.champion.brain.genes,
            "name": self.champion.name,
            "generation": self.generation,
            "race": self.race_number,
            "streak": self.streak,
            "record": self.record,
        }

        with open("ai_racer_champion.json", "w", encoding="utf-8") as f:
            json.dump(data, f)

    def load(self):
        if not os.path.exists("ai_racer_champion.json"):
            return False

        with open("ai_racer_champion.json", "r", encoding="utf-8") as f:
            data = json.load(f)

        self.champion = Car(
            data.get("name", "CHAMPION-LOADED"),
            Brain(data["genes"]),
            COLORS[0]
        )

        self.generation = int(data.get("generation", 1))
        self.streak = int(data.get("streak", 0))
        self.record = int(data.get("record", 0))
        self.race_number = int(data.get("race", 0))

        self.new_race()
        return True


class App:
    def __init__(self, root):
        self.root = root
        self.root.title("AI Racer 4")
        self.root.configure(bg=BG)
        self.root.resizable(False, False)

        self.game = Tournament()

        self.running = False
        self.fast = True
        self.steps = 300

        self.canvas = tk.Canvas(
            root,
            width=WIDTH,
            height=HEIGHT,
            bg=BG,
            highlightthickness=0
        )
        self.canvas.pack(padx=10, pady=10)

        panel = tk.Frame(root, bg="#181d24")
        panel.pack(fill="x", padx=10, pady=(0, 10))

        tk.Button(
            panel,
            text="START / STOP",
            command=self.start_stop,
            width=13
        ).pack(side="left", padx=4, pady=6)

        tk.Button(
            panel,
            text="SPEED",
            command=self.change_speed,
            width=10
        ).pack(side="left", padx=4)

        tk.Button(
            panel,
            text="NEW RACE",
            command=self.new_race,
            width=10
        ).pack(side="left", padx=4)

        tk.Button(
            panel,
            text="SAVE",
            command=self.save,
            width=8
        ).pack(side="left", padx=4)

        tk.Button(
            panel,
            text="LOAD",
            command=self.load,
            width=8
        ).pack(side="left", padx=4)

        self.label = tk.Label(
            panel,
            text="",
            bg="#181d24",
            fg=WHITE,
            justify="left",
            font=("Consolas", 10)
        )
        self.label.pack(side="left", padx=10)

        self.draw()
        self.update_label()

    def point(self, pos):
        angle = pos * math.pi * 2

        x = CX + RX * math.cos(angle)
        y = CY + RY * math.sin(angle)

        return x, y

    def tangent_angle(self, pos):
        angle = pos * math.pi * 2

        dx = -RX * math.sin(angle)
        dy = RY * math.cos(angle)

        return math.atan2(dy, dx)

    def draw(self):
        c = self.canvas
        c.delete("all")

        # Фон.
        c.create_rectangle(
            0, 0, WIDTH, HEIGHT,
            fill=GREEN,
            outline=""
        )

        # Дорога.
        c.create_oval(
            CX - RX,
            CY - RY,
            CX + RX,
            CY + RY,
            outline=ROAD,
            width=120
        )

        # Центральная линия.
        c.create_oval(
            CX - RX,
            CY - RY,
            CX + RX,
            CY + RY,
            outline=WHITE,
            width=2,
            dash=(15, 18)
        )

        # Старт.
        c.create_line(
            CX,
            CY - RY - 55,
            CX,
            CY - RY + 55,
            fill=WHITE,
            width=5
        )

        # Машины.
        ordered = sorted(
            self.game.cars,
            key=lambda z: (z.lap, z.pos),
            reverse=True
        )

        for rank, car in enumerate(ordered, 1):
            x, y = self.point(car.pos)
            angle = self.tangent_angle(car.pos)

            c.create_rectangle(
                x - CAR_W / 2,
                y - CAR_H / 2,
                x + CAR_W / 2,
                y + CAR_H / 2,
                fill=car.color,
                outline=""
            )

            tx = x + math.cos(angle) * 17
            ty = y + math.sin(angle) * 17

            c.create_line(
                x, y, tx, ty,
                fill=WHITE,
                width=3
            )

            c.create_text(
                x,
                y - 22,
                text="%d. %s" % (rank, car.name),
                fill=WHITE,
                font=("Arial", 8, "bold")
            )

        c.create_text(
            15,
            15,
            anchor="nw",
            text="RACE %d" % self.game.race_number,
            fill=WHITE,
            font=("Arial", 13, "bold")
        )

        c.create_text(
            WIDTH - 15,
            15,
            anchor="ne",
            text="CHAMPION: %s" % self.game.champion.name,
            fill=COLORS[0],
            font=("Arial", 11, "bold")
        )

        c.create_text(
            WIDTH - 15,
            38,
            anchor="ne",
            text="WIN STREAK: %d" % self.game.streak,
            fill=WHITE,
            font=("Arial", 10)
        )

    def update_label(self):
        champion = self.game.champion

        text = (
            "Generation: %d    Race: %d    Cars: 4\n"
            "Champion wins: %d    Record: %d\n"
            "Mode: %s"
        ) % (
            self.game.generation,
            self.game.race_number,
            self.game.streak,
            self.game.record,
            "FAST" if self.fast else "NORMAL"
        )

        self.label.config(text=text)

    def start_stop(self):
        self.running = not self.running

        if self.running:
            self.loop()

    def change_speed(self):
        self.fast = not self.fast
        self.steps = 300 if self.fast else 1
        self.update_label()

    def new_race(self):
        self.running = False
        self.game.new_race()
        self.draw()
        self.update_label()

    def save(self):
        try:
            self.game.save()
        except Exception as e:
            print("Save error:", e)

    def load(self):
        try:
            if self.game.load():
                self.draw()
                self.update_label()
        except Exception as e:
            print("Load error:", e)

    def loop(self):
        if not self.running:
            return

        for _ in range(self.steps):
            result = self.game.step()

            if result is not None:
                winner, kept = result

                if kept:
                    print("Champion wins:", winner)
                else:
                    print("NEW CHAMPION:", winner)

        self.draw()
        self.update_label()

        delay = 1 if self.fast else 20
        self.root.after(delay, self.loop)


def main():
    root = tk.Tk()
    App(root)
    root.mainloop()


if __name__ == "__main__":
    main()
