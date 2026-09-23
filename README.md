# -*- coding: utf-8 -*-
"""
Симулятор валют / экономики для Pydroid 3 (Kivy)
Один файл. Сохранение в currencies.json рядом со скриптом.
v3: без дефолтных валют + объединение валют.
"""

import os
import sys
import json
import math
import random
import colorsys
import traceback

try:
    BASE_DIR = os.path.dirname(os.path.abspath(__file__))
except Exception:
    BASE_DIR = os.getcwd()

if not BASE_DIR or not os.path.isdir(BASE_DIR):
    BASE_DIR = os.getcwd()

DATA_FILE = os.path.join(BASE_DIR, "currencies.json")
ERROR_LOG = os.path.join(BASE_DIR, "error.log")


def log_error():
    try:
        with open(ERROR_LOG, "a", encoding="utf-8") as f:
            f.write("=" * 60 + "\n")
            f.write(traceback.format_exc())
            f.write("\n")
    except Exception:
        pass


try:
    from kivy.app import App
    from kivy.core.window import Window
    from kivy.uix.screenmanager import ScreenManager, Screen
    from kivy.uix.boxlayout import BoxLayout
    from kivy.uix.scrollview import ScrollView
    from kivy.uix.label import Label
    from kivy.uix.button import Button
    from kivy.uix.textinput import TextInput
    from kivy.uix.widget import Widget
    from kivy.graphics import Color, Rectangle, Line
    from kivy.metrics import dp
except Exception:
    log_error()
    raise

Window.clearcolor = (0, 0, 0, 1)


# ---------- ПАЛИТРА ----------
def make_palette():
    pal = []
    for i in range(100):
        h = (i / 100.0) % 1.0
        r, g, b = colorsys.hsv_to_rgb(h, 0.75, 0.95)
        pal.append((round(r, 3), round(g, 3), round(b, 3), 1.0))
    return pal

PALETTE = make_palette()


# ---------- МОДЕЛЬ ----------
class Currency:
    def __init__(self, code, name, emoji, color, rate):
        self.code = code
        self.name = name
        self.emoji = emoji
        self.color = list(color)
        self.rate = float(rate)
        self.history = [float(rate)]
        self.effects = []  # [{"name": str, "delta": float, "remaining": int}]

    def to_dict(self):
        return {
            "code": self.code,
            "name": self.name,
            "emoji": self.emoji,
            "color": self.color,
            "rate": self.rate,
            "history": self.history,
            "effects": self.effects,
        }

    @classmethod
    def from_dict(cls, d):
        c = cls(d["code"], d["name"], d.get("emoji", ""),
                d["color"], d.get("rate", 1.0))
        c.history = list(d.get("history", [c.rate]))
        c.effects = list(d.get("effects", []))
        return c


class GameState:
    def __init__(self):
        self.currencies = []
        self.turn = 0

    def save(self):
        try:
            with open(DATA_FILE, "w", encoding="utf-8") as f:
                json.dump({
                    "turn": self.turn,
                    "currencies": [c.to_dict() for c in self.currencies],
                }, f, ensure_ascii=False, indent=2)
            return True
        except Exception:
            log_error()
            return False

    def load(self):
        if not os.path.exists(DATA_FILE):
            return False
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            self.turn = int(data.get("turn", 0))
            self.currencies = [Currency.from_dict(d) for d in data.get("currencies", [])]
            return len(self.currencies) > 0
        except Exception:
            log_error()
            return False


# ---------- РАЗОВЫЕ ИВЕНТЫ ----------
EVENTS = {
    "ВОЙНА": [
        ("Малая война", -0.4),
        ("Средняя война", -1.0),
        ("Большая война", -5.0),
        ("Война сверхдержав", -10.0),
        ("Гражданская война", -15.0),
        ("Мировая война", -30.0),
        ("Победа в войне", 5.0),
        ("Военный переворот", -7.0),
        ("Партизанская война", -2.0),
        ("Ядерный удар", -50.0),
    ],
    "БЛОКАДА": [
        ("Малая блокада", -0.1),
        ("Средняя блокада", -0.5),
        ("Большая блокада", -2.5),
        ("Блокада сверхдержавы", -5.0),
        ("Мировая блокада", -100.0),
        ("Снятие блокады", 3.0),
        ("Торговое эмбарго", -4.0),
    ],
    "ТОРГОВЛЯ": [
        ("Малая торговая сделка", 0.1),
        ("Средняя торговая сделка", 0.25),
        ("Большая торговая сделка", 1.0),
        ("Торговый союз", 3.0),
        ("Экспортный бум", 5.0),
        ("Открытие рынков", 2.0),
        ("Повышение тарифов", -1.5),
        ("Торговая война", -6.0),
        ("Новый торговый путь", 1.5),
    ],
    "ТЕХНОЛОГИИ": [
        ("Технологический прорыв", 5.0),
        ("Прорыв в ИИ", 6.0),
        ("Кибератака", -3.0),
        ("Массовая кибератака", -8.0),
        ("Новый материал", 3.0),
        ("Космическая программа", 2.0),
        ("Утечка технологий", -2.0),
        ("Квантовый прорыв", 8.0),
    ],
    "ЭКОНОМИКА": [
        ("Экономический бум", 5.0),
        ("Инфляция", -2.0),
        ("Гиперинфляция", -10.0),
        ("Кризис", -8.0),
        ("Дефолт", -20.0),
        ("Реформа", 2.0),
        ("Стимул экономики", 1.5),
        ("Приватизация", 1.0),
        ("Банковский крах", -12.0),
        ("Снижение ставки", 0.5),
        ("Повышение ставки", -0.5),
        ("Рост ВВП", 3.0),
    ],
    "ПОЛИТИКА": [
        ("Выборы", 0.2),
        ("Смена власти", -1.0),
        ("Стабильность", 0.5),
        ("Коррупционный скандал", -2.0),
        ("Референдум", 0.3),
        ("Политический кризис", -5.0),
        ("Дипломатический успех", 2.0),
        ("Санкции", -7.0),
        ("Снятие санкций", 4.0),
    ],
    "ПРИРОДА": [
        ("Землетрясение", -3.0),
        ("Наводнение", -2.0),
        ("Засуха", -2.5),
        ("Ураган", -1.5),
        ("Эпидемия", -6.0),
        ("Пандемия", -15.0),
        ("Богатый урожай", 2.0),
        ("Открытие месторождения", 5.0),
        ("Извержение вулкана", -4.0),
    ],
    "ДРУГОЕ": [
        ("Случайный рост", 1.0),
        ("Случайное падение", -1.0),
        ("Спекуляции", 2.0),
        ("Обвал рынка", -10.0),
        ("Позитивные новости", 1.5),
        ("Негативные новости", -1.5),
        ("Загадочное событие", 0.0),
        ("Нейтральное событие", 0.0),
    ],
}


# ---------- ПАССИВНЫЕ ЭФФЕКТЫ ----------
ONGOING_EFFECTS = [
    ("Лёгкая инфляция",            -0.01, 10, "ЭКОНОМИКА"),
    ("Слабый рост",                +0.01, 10, "ЭКОНОМИКА"),
    ("Стагнация",                  -0.02,  8, "ЭКОНОМИКА"),
    ("Экономический рост",         +0.05, 10, "ЭКОНОМИКА"),
    ("Утечка капитала",            -0.05, 12, "ЭКОНОМИКА"),
    ("Восстановление экономики",   +0.03, 15, "ЭКОНОМИКА"),
    ("Инвестиции в инфраструктуру", +0.04, 12, "ЭКОНОМИКА"),
    ("Льготные кредиты",           +0.03,  6, "ЭКОНОМИКА"),
    ("Галопирующая инфляция",      -0.15,  8, "ЭКОНОМИКА"),
    ("Коррупция",                  -0.04, 20, "ПОЛИТИКА"),
    ("Санкционное давление",       -0.08, 15, "ПОЛИТИКА"),
    ("Образовательная реформа",    +0.01, 20, "ПОЛИТИКА"),
    ("Забастовки",                 -0.05,  5, "ПОЛИТИКА"),
    ("Политическая стабильность",  +0.02, -1, "ПОЛИТИКА"),
    ("Военные расходы",            -0.10, 10, "ВОЙНА"),
    ("Партизанская активность",    -0.06, 12, "ВОЙНА"),
    ("Гонка вооружений",           -0.08, -1, "ВОЙНА"),
    ("Туристический бум",          +0.06,  8, "ТОРГОВЛЯ"),
    ("Торговое соглашение",        +0.05, 10, "ТОРГОВЛЯ"),
    ("Эмбарго на экспорт",         -0.07, 10, "ТОРГОВЛЯ"),
    ("Технологическое преимущество", +0.02, -1, "ТЕХНОЛОГИИ"),
    ("Кибершпионаж",               -0.04,  8, "ТЕХНОЛОГИИ"),
    ("Прорыв в энергетике",        +0.06, 10, "ТЕХНОЛОГИИ"),
    ("Демографический спад",       -0.02, -1, "ПРИРОДА"),
    ("Старение населения",         -0.03, -1, "ПРИРОДА"),
    ("Загрязнение",                -0.02, -1, "ПРИРОДА"),
    ("Валютные спекуляции",        -0.03,  6, "ДРУГОЕ"),
    ("Культурный обмен",           +0.01,  8, "ДРУГОЕ"),
]


# ---------- ГРАФИК ЛИНИЙ ----------
class LineChart(Widget):
    def __init__(self, state, **kwargs):
        super().__init__(**kwargs)
        self.state = state
        self.bind(pos=self.redraw, size=self.redraw)

    def redraw(self, *args):
        try:
            self.canvas.clear()
            if self.width <= 1 or self.height <= 1:
                return
            with self.canvas:
                Color(0.04, 0.04, 0.04, 1)
                Rectangle(pos=self.pos, size=self.size)
                Color(0.15, 0.15, 0.15, 1)
                for i in range(1, 5):
                    y = self.y + self.height * i / 5.0
                    Line(points=[self.x, y, self.x + self.width, y], width=1)

            if not self.state.currencies:
                return
            max_len = max(len(c.history) for c in self.state.currencies)
            if max_len < 2:
                return
            all_vals = []
            for c in self.state.currencies:
                for v in c.history[-max_len:]:
                    if v > 0:
                        all_vals.append(v)
            if not all_vals:
                return
            lmin = math.log10(max(min(all_vals), 1e-6))
            lmax = math.log10(max(all_vals))
            if lmax - lmin < 1e-6:
                lmax = lmin + 1

            with self.canvas:
                for c in self.state.currencies:
                    hist = c.history[-max_len:]
                    n = len(hist)
                    if n < 2:
                        continue
                    Color(c.color[0], c.color[1], c.color[2], 1)
                    pts = []
                    for i, v in enumerate(hist):
                        x = self.x + self.width * i / (n - 1)
                        lv = math.log10(max(v, 1e-6))
                        y = self.y + self.height * (lv - lmin) / (lmax - lmin)
                        pts.extend([x, y])
                    Line(points=pts, width=1.6)
        except Exception:
            log_error()


class BarChart(Widget):
    def __init__(self, currency, **kwargs):
        super().__init__(**kwargs)
        self.currency = currency
        self.bind(pos=self.redraw, size=self.redraw)

    def set_currency(self, c):
        self.currency = c
        self.redraw()

    def redraw(self, *args):
        try:
            self.canvas.clear()
            if self.width <= 1 or self.height <= 1:
                return
            with self.canvas:
                Color(0.04, 0.04, 0.04, 1)
                Rectangle(pos=self.pos, size=self.size)
            c = self.currency
            if not c:
                return
            hist = c.history[-5:]
            if not hist:
                return
            hmax = max(hist) if max(hist) > 0 else 1.0
            n = len(hist)
            gap = dp(8)
            bottom_pad = dp(10)
            top_pad = dp(10)
            avail_h = self.height - bottom_pad - top_pad
            bw = max(1.0, (self.width - gap * (n + 1)) / n)
            with self.canvas:
                for i, v in enumerate(hist):
                    h = avail_h * (v / hmax) if hmax > 0 else 0
                    x = self.x + gap + i * (bw + gap)
                    y = self.y + bottom_pad
                    Color(c.color[0], c.color[1], c.color[2], 1)
                    Rectangle(pos=(x, y), size=(bw, max(1.0, h)))
        except Exception:
            log_error()


# ---------- ГЛАВНЫЙ ЭКРАН ----------
class MainScreen(Screen):
    def __init__(self, app, **kwargs):
        super().__init__(**kwargs)
        self.app = app
        self._build()

    def _build(self):
        root = BoxLayout(orientation="vertical")
        self.chart = LineChart(self.app.state, size_hint_y=None, height=dp(190))
        root.add_widget(self.chart)

        sv = ScrollView()
        self.list_layout = BoxLayout(
            orientation="vertical", size_hint_y=None,
            spacing=dp(4), padding=dp(6),
        )
        self.list_layout.bind(minimum_height=self.list_layout.setter("height"))
        sv.add_widget(self.list_layout)
        root.add_widget(sv)

        row1 = BoxLayout(size_hint_y=None, height=dp(50),
                         spacing=dp(4), padding=dp(4))
        b_add = Button(text="+ Создать")
        b_add.bind(on_release=self._go_create)
        row1.add_widget(b_add)
        b_merge = Button(text="Объединить")
        b_merge.bind(on_release=self._go_merge)
        row1.add_widget(b_merge)
        root.add_widget(row1)

        row2 = BoxLayout(size_hint_y=None, height=dp(50),
                         spacing=dp(4), padding=dp(4))
        b_next = Button(text="Следующий ход")
        b_next.bind(on_release=self._next_turn)
        row2.add_widget(b_next)
        b_reset = Button(text="Сброс")
        b_reset.bind(on_release=self._go_reset)
        row2.add_widget(b_reset)
        root.add_widget(row2)

        self.add_widget(root)

    def on_enter(self):
        self.refresh()

    def refresh(self):
        try:
            self.chart.redraw()
            self.list_layout.clear_widgets()
            if not self.app.state.currencies:
                lbl = Label(text="Нет валют. Нажмите «+ Создать».",
                            size_hint_y=None, height=dp(60))
                lbl.color = (0.7, 0.7, 0.7, 1)
                self.list_layout.add_widget(lbl)
                return

            for c in self.app.state.currencies:
                chg = 0.0
                if len(c.history) >= 2 and c.history[-2] > 0:
                    chg = (c.history[-1] - c.history[-2]) / c.history[-2] * 100.0
                sign = "+" if chg >= 0 else ""
                eff_count = len(c.effects)
                eff_txt = "   [{} эфф.]".format(eff_count) if eff_count else ""
                text = "{}  {}  {}\n{:.4f}   {}{:.2f}%{}".format(
                    c.emoji, c.code, c.name, c.rate, sign, chg, eff_txt
                )
                b = Button(text=text, size_hint_y=None, height=dp(74))
                b.background_normal = ""
                b.background_color = (0.10, 0.10, 0.10, 1)
                if chg > 0:
                    b.color = (0.35, 1.0, 0.35, 1)
                elif chg < 0:
                    b.color = (1.0, 0.35, 0.35, 1)
                else:
                    b.color = (0.85, 0.85, 0.85, 1)
                b.bind(on_release=lambda inst, cc=c: self.app.open_detail(cc))
                self.list_layout.add_widget(b)
        except Exception:
            log_error()

    def _go_create(self, *a):
        self.app.sm.current = "create"

    def _go_merge(self, *a):
        self.app.sm.current = "merge"

    def _go_reset(self, *a):
        self.app.sm.current = "reset"

    def _next_turn(self, *a):
        try:
            for c in self.app.state.currencies:
                delta = random.uniform(-0.05, 0.05)
                c.rate *= (1.0 + delta)

                total_pct = 0.0
                new_effects = []
                for eff in c.effects:
                    total_pct += float(eff.get("delta", 0.0))
                    rem = int(eff.get("remaining", 0))
                    if rem > 0:
                        eff["remaining"] = rem - 1
                        if eff["remaining"] > 0:
                            new_effects.append(eff)
                    else:
                        new_effects.append(eff)
                c.effects = new_effects

                c.rate *= (1.0 + total_pct / 100.0)
                c.rate = max(0.01, c.rate)
                c.history.append(c.rate)

            self.app.state.turn += 1
            self.app.state.save()
            self.refresh()
        except Exception:
            log_error()


# ---------- ЭКРАН СОЗДАНИЯ ----------
class CreateScreen(Screen):
    def __init__(self, app, **kwargs):
        super().__init__(**kwargs)
        self.app = app
        self.selected_color = list(PALETTE[0])
        self._build()

    def _build(self):
        root = BoxLayout(orientation="vertical", padding=dp(8), spacing=dp(8))

        back = Button(text="← Назад", size_hint_y=None, height=dp(44))
        back.bind(on_release=lambda *a: setattr(self.app.sm, "current", "main"))
        root.add_widget(back)

        title = Label(text="Новая валюта", size_hint_y=None, height=dp(34))
        title.color = (1, 1, 1, 1)
        root.add_widget(title)

        self.ti_emoji = TextInput(hint_text="Эмодзи/флаг (опц.)",
                                  size_hint_y=None, height=dp(44), multiline=False)
        root.add_widget(self.ti_emoji)
        self.ti_code = TextInput(hint_text="Код (USD)",
                                 size_hint_y=None, height=dp(44), multiline=False)
        root.add_widget(self.ti_code)
        self.ti_name = TextInput(hint_text="Название (Доллар)",
                                 size_hint_y=None, height=dp(44), multiline=False)
        root.add_widget(self.ti_name)
        self.ti_rate = TextInput(text="100",
                                 size_hint_y=None, height=dp(44), multiline=False)
        root.add_widget(self.ti_rate)

        lbl_c = Label(text="Цвет (100 вариантов)", size_hint_y=None, height=dp(28))
        lbl_c.color = (0.8, 0.8, 0.8, 1)
        root.add_widget(lbl_c)

        sv = ScrollView(size_hint_y=None, height=dp(60),
                        do_scroll_y=False, do_scroll_x=True)
        pal_box = BoxLayout(orientation="horizontal",
                            size_hint_x=None, spacing=dp(4))
        pal_box.bind(minimum_width=pal_box.setter("width"))
        for col in PALETTE:
            b = Button(size_hint=(None, 1), width=dp(44))
            b.background_normal = ""
            b.background_color = col
            b.color = (0, 0, 0, 0)
            b.bind(on_release=lambda inst, c=col: self._pick(c))
            pal_box.add_widget(b)
        sv.add_widget(pal_box)
        root.add_widget(sv)

        self.status = Label(text="", size_hint_y=None, height=dp(28))
        self.status.color = (1, 0.6, 0.4, 1)
        root.add_widget(self.status)

        create = Button(text="Создать", size_hint_y=None, height=dp(50))
        create.bind(on_release=self._create)
        root.add_widget(create)

        self.add_widget(root)

    def _pick(self, c):
        self.selected_color = list(c)

    def _create(self, *a):
        try:
            code = (self.ti_code.text or "").strip().upper()
            name = (self.ti_name.text or "").strip()
            emoji = (self.ti_emoji.text or "").strip()
            try:
                rate = float((self.ti_rate.text or "").strip())
            except Exception:
                rate = 100.0
            if rate <= 0:
                rate = 100.0

            if not code:
                self.status.text = "Введите код валюты."
                return
            if not name:
                self.status.text = "Введите название."
                return
            if any(c.code == code for c in self.app.state.currencies):
                self.status.text = "Такой код уже существует."
                return

            self.app.state.currencies.append(
                Currency(code, name, emoji, self.selected_color, rate)
            )
            self.app.state.save()
            self.ti_code.text = ""
            self.ti_name.text = ""
            self.ti_emoji.text = ""
            self.ti_rate.text = "100"
            self.status.text = ""
            self.app.sm.current = "main"
        except Exception:
            log_error()
            self.status.text = "Ошибка создания (см. error.log)"


# ---------- ЭКРАН ОБЪЕДИНЕНИЯ ----------
class MergeScreen(Screen):
    def __init__(self, app, **kwargs):
        super().__init__(**kwargs)
        self.app = app
        self.selected = set()
        self.selected_color = list(PALETTE[0])
        self._build()

    def _build(self):
        root = BoxLayout(orientation="vertical", padding=dp(8), spacing=dp(6))

        back = Button(text="← Назад", size_hint_y=None, height=dp(44))
        back.bind(on_release=lambda *a: self.app.go_main())
        root.add_widget(back)

        title = Label(text="Объединение валют", size_hint_y=None, height=dp(30))
        title.color = (1, 1, 1, 1)
        root.add_widget(title)

        hint = Label(text="Выберите 2+ валюты для слияния", size_hint_y=None,
                     height=dp(22))
        hint.font_size = dp(12)
        hint.color = (0.7, 0.7, 0.7, 1)
        root.add_widget(hint)

        self.ti_code = TextInput(hint_text="Новый код (опц.)",
                                 size_hint_y=None, height=dp(42), multiline=False)
        root.add_widget(self.ti_code)
        self.ti_name = TextInput(hint_text="Название (опц.)",
                                 size_hint_y=None, height=dp(42), multiline=False)
        root.add_widget(self.ti_name)
        self.ti_emoji = TextInput(hint_text="Эмодзи/флаг (опц.)",
                                  size_hint_y=None, height=dp(42), multiline=False)
        root.add_widget(self.ti_emoji)

        sv = ScrollView()
        self.list_box = BoxLayout(orientation="vertical",
                                  size_hint_y=None, spacing=dp(2))
        self.list_box.bind(minimum_height=self.list_box.setter("height"))
        sv.add_widget(self.list_box)
        root.add_widget(sv)

        lbl_c = Label(text="Цвет новой валюты", size_hint_y=None, height=dp(22))
        lbl_c.font_size = dp(12)
        lbl_c.color = (0.8, 0.8, 0.8, 1)
        root.add_widget(lbl_c)

        svc = ScrollView(size_hint_y=None, height=dp(44),
                         do_scroll_y=False, do_scroll_x=True)
        pal_box = BoxLayout(orientation="horizontal",
                            size_hint_x=None, spacing=dp(3))
        pal_box.bind(minimum_width=pal_box.setter("width"))
        for col in PALETTE:
            b = Button(size_hint=(None, 1), width=dp(38))
            b.background_normal = ""
            b.background_color = col
            b.color = (0, 0, 0, 0)
            b.bind(on_release=lambda inst, c=col: self._pick(c))
            pal_box.add_widget(b)
        svc.add_widget(pal_box)
        root.add_widget(svc)

        self.status = Label(text="", size_hint_y=None, height=dp(22))
        self.status.font_size = dp(12)
        self.status.color = (1, 0.6, 0.4, 1)
        root.add_widget(self.status)

        merge_btn = Button(text="Объединить", size_hint_y=None, height=dp(50))
        merge_btn.bind(on_release=self._merge)
        root.add_widget(merge_btn)

        self.add_widget(root)

    def on_enter(self):
        self.selected = set()
        self._refresh_list()
        self._update_status()

    def _refresh_list(self):
        self.list_box.clear_widgets()
        if not self.app.state.currencies:
            lbl = Label(text="Нет валют для объединения.",
                        size_hint_y=None, height=dp(50))
            lbl.color = (0.7, 0.7, 0.7, 1)
            self.list_box.add_widget(lbl)
            return
        for c in self.app.state.currencies:
            mark = "✓" if c.code in self.selected else "○"
            text = "{}  {}  {}  —  {:.2f}".format(mark, c.emoji, c.code, c.rate)
            b = Button(text=text, size_hint_y=None, height=dp(46))
            b.background_normal = ""
            if c.code in self.selected:
                b.background_color = (0.15, 0.30, 0.15, 1)
                b.color = (0.5, 1.0, 0.5, 1)
            else:
                b.background_color = (0.10, 0.10, 0.10, 1)
                b.color = (0.85, 0.85, 0.85, 1)
            b.bind(on_release=lambda inst, cc=c.code: self._toggle(cc))
            self.list_box.add_widget(b)

    def _toggle(self, code):
        if code in self.selected:
            self.selected.discard(code)
        else:
            self.selected.add(code)
        self._refresh_list()
        self._update_status()

    def _update_status(self):
        n = len(self.selected)
        if n == 0:
            self.status.text = "Ничего не выбрано"
        else:
            self.status.text = "Выбрано: {} валют".format(n)

    def _pick(self, c):
        self.selected_color = list(c)

    def _merge(self, *a):
        try:
            if len(self.selected) < 2:
                self.status.text = "Выберите минимум 2 валюты"
                return

            code = (self.ti_code.text or "").strip().upper()
            name = (self.ti_name.text or "").strip()
            emoji = (self.ti_emoji.text or "").strip()

            selected_currs = [c for c in self.app.state.currencies
                              if c.code in self.selected]
            if len(selected_currs) < 2:
                self.status.text = "Валюты не найдены"
                return

            if not code:
                code = "+".join([c.code for c in selected_currs])[:8]
            if not name:
                name = " + ".join([c.name for c in selected_currs])[:30]
            if not emoji:
                emojis = [c.emoji for c in selected_currs if c.emoji]
                emoji = "".join(emojis)[:4]

            # проверка на дубликат кода среди НЕ выбранных
            if any(c.code == code for c in self.app.state.currencies
                   if c.code not in self.selected):
                self.status.text = "Код уже существует"
                return

            # новый курс = среднее курсов
            new_rate = sum(c.rate for c in selected_currs) / len(selected_currs)

            # история: усреднение с выравниванием от конца
            max_len = max(len(c.history) for c in selected_currs)
            new_history = []
            for i in range(max_len):
                vals = []
                for c in selected_currs:
                    idx = len(c.history) - max_len + i
                    if 0 <= idx < len(c.history):
                        vals.append(c.history[idx])
                new_history.append(sum(vals) / len(vals) if vals else new_rate)
            if not new_history:
                new_history = [new_rate]

            # эффекты: суммируем по имени, берём макс. длительность
            combined = {}
            for c in selected_currs:
                for eff in c.effects:
                    key = eff.get("name", "?")
                    d = float(eff.get("delta", 0.0))
                    r = int(eff.get("remaining", 0))
                    if key in combined:
                        combined[key]["delta"] = float(combined[key]["delta"]) + d
                        prev_r = combined[key]["remaining"]
                        if prev_r <= 0 or r <= 0:
                            # если хоть один постоянный → постоянный
                            if prev_r <= 0 or r <= 0:
                                combined[key]["remaining"] = -1
                        else:
                            combined[key]["remaining"] = max(prev_r, r)
                    else:
                        combined[key] = {"name": key, "delta": d, "remaining": r}

            new_curr = Currency(code, name, emoji, self.selected_color, new_rate)
            new_curr.history = new_history
            new_curr.effects = list(combined.values())

            # удаляем выбранные, добавляем новую
            self.app.state.currencies = [
                c for c in self.app.state.currencies
                if c.code not in self.selected
            ]
            self.app.state.currencies.append(new_curr)
            self.app.state.save()

            self.ti_code.text = ""
            self.ti_name.text = ""
            self.ti_emoji.text = ""
            self.selected = set()
            self.app.go_main()
        except Exception:
            log_error()
            self.status.text = "Ошибка (см. error.log)"


# ---------- ЭКРАН ДЕТАЛЕЙ ----------
class DetailScreen(Screen):
    def __init__(self, app, **kwargs):
        super().__init__(**kwargs)
        self.app = app
        self.currency = None
        self._build()

    def _build(self):
        root = BoxLayout(orientation="vertical", padding=dp(10), spacing=dp(6))

        back = Button(text="← Назад", size_hint_y=None, height=dp(44))
        back.bind(on_release=lambda *a: setattr(self.app.sm, "current", "main"))
        root.add_widget(back)

        self.lbl_code = Label(text="", size_hint_y=None, height=dp(60))
        self.lbl_code.font_size = dp(36)
        root.add_widget(self.lbl_code)

        self.lbl_name = Label(text="", size_hint_y=None, height=dp(34))
        self.lbl_name.font_size = dp(20)
        self.lbl_name.color = (0.85, 0.85, 0.85, 1)
        root.add_widget(self.lbl_name)

        self.lbl_rate = Label(text="", size_hint_y=None, height=dp(50))
        self.lbl_rate.font_size = dp(30)
        self.lbl_rate.color = (1, 1, 1, 1)
        root.add_widget(self.lbl_rate)

        self.lbl_change = Label(text="", size_hint_y=None, height=dp(34))
        self.lbl_change.font_size = dp(20)
        root.add_widget(self.lbl_change)

        self.lbl_eff = Label(text="", size_hint_y=None, height=dp(24))
        self.lbl_eff.font_size = dp(13)
        self.lbl_eff.color = (1.0, 0.8, 0.4, 1)
        root.add_widget(self.lbl_eff)

        ev = Button(text="Ивенты и эффекты", size_hint_y=None, height=dp(50))
        ev.bind(on_release=self._open_events)
        root.add_widget(ev)

        lbl_h = Label(text="Последние 5 ходов", size_hint_y=None, height=dp(26))
        lbl_h.color = (0.7, 0.7, 0.7, 1)
        root.add_widget(lbl_h)

        self.chart = BarChart(None, size_hint_y=None, height=dp(220))
        root.add_widget(self.chart)

        self.add_widget(root)

    def set_currency(self, c):
        self.currency = c
        self.refresh()

    def on_enter(self):
        self.refresh()

    def refresh(self):
        try:
            c = self.currency
            if not c:
                return
            self.lbl_code.text = "{} {}".format(c.emoji, c.code)
            self.lbl_code.color = (c.color[0], c.color[1], c.color[2], 1)
            self.lbl_name.text = c.name
            self.lbl_rate.text = "{:.4f}".format(c.rate)

            if len(c.history) >= 2 and c.history[-2] > 0:
                chg = (c.history[-1] - c.history[-2]) / c.history[-2] * 100.0
                self.lbl_change.text = "{:+.2f}%".format(chg)
                if chg > 0:
                    self.lbl_change.color = (0.35, 1.0, 0.35, 1)
                elif chg < 0:
                    self.lbl_change.color = (1.0, 0.35, 0.35, 1)
                else:
                    self.lbl_change.color = (0.85, 0.85, 0.85, 1)
            else:
                self.lbl_change.text = "0.00%"
                self.lbl_change.color = (0.85, 0.85, 0.85, 1)

            n = len(c.effects)
            if n == 0:
                self.lbl_eff.text = "Пассивных эффектов нет"
            else:
                total = sum(float(e.get("delta", 0.0)) for e in c.effects)
                self.lbl_eff.text = "Активно эффектов: {}  (сумма {:+.2f}%/ход)".format(
                    n, total
                )

            self.chart.set_currency(c)
        except Exception:
            log_error()

    def _open_events(self, *a):
        if self.currency:
            self.app.open_events(self.currency)


# ---------- ЭКРАН ИВЕНТОВ ----------
class EventsScreen(Screen):
    def __init__(self, app, **kwargs):
        super().__init__(**kwargs)
        self.app = app
        self.currency = None
        self._build()

    def _build(self):
        root = BoxLayout(orientation="vertical", padding=dp(8), spacing=dp(4))

        back = Button(text="← Назад", size_hint_y=None, height=dp(44))
        back.bind(on_release=self._back)
        root.add_widget(back)

        self.title_lbl = Label(text="Ивенты", size_hint_y=None, height=dp(34))
        self.title_lbl.color = (1, 1, 1, 1)
        root.add_widget(self.title_lbl)

        sv = ScrollView()
        self.cont = BoxLayout(orientation="vertical",
                              size_hint_y=None, spacing=dp(2))
        self.cont.bind(minimum_height=self.cont.setter("height"))
        sv.add_widget(self.cont)
        root.add_widget(sv)

        self.add_widget(root)

    def set_currency(self, c):
        self.currency = c
        self._rebuild()

    def on_enter(self):
        self._rebuild()

    def _rebuild(self):
        try:
            self.cont.clear_widgets()
            c = self.currency
            if not c:
                return
            self.title_lbl.text = "{} {}".format(c.emoji, c.code)

            if c.effects:
                hdr = Label(
                    text="АКТИВНЫЕ ЭФФЕКТЫ ({}): тап — убрать".format(len(c.effects)),
                    size_hint_y=None, height=dp(28))
                hdr.color = (1.0, 0.85, 0.4, 1)
                self.cont.add_widget(hdr)
                for i, eff in enumerate(c.effects):
                    rem = eff.get("remaining", 0)
                    rem_txt = "∞" if rem <= 0 else "{}х".format(rem)
                    d = float(eff.get("delta", 0.0))
                    text = "{}   {:+.2f}%/ход  ({})".format(
                        eff.get("name", "?"), d, rem_txt)
                    b = Button(text=text, size_hint_y=None, height=dp(42))
                    b.background_normal = ""
                    b.background_color = (0.18, 0.10, 0.10, 1)
                    b.color = (1.0, 0.7, 0.7, 1)
                    b.bind(on_release=lambda inst, idx=i: self._remove(idx))
                    self.cont.add_widget(b)

            hdr2 = Label(text="— ПОСТОЯННЫЕ ЭФФЕКТЫ (за ход) —",
                         size_hint_y=None, height=dp(30))
            hdr2.color = (1.0, 0.8, 0.3, 1)
            self.cont.add_widget(hdr2)

            for name, delta, duration, cat in ONGOING_EFFECTS:
                rem_txt = "∞" if duration < 0 else "{}х".format(duration)
                sign = "+" if delta >= 0 else ""
                text = "{}  [{}{:.2f}%/ход, {}]  ·  {}".format(
                    name, sign, delta, rem_txt, cat)
                b = Button(text=text, size_hint_y=None, height=dp(42))
                b.background_normal = ""
                if delta > 0:
                    b.color = (0.35, 1.0, 0.35, 1)
                elif delta < 0:
                    b.color = (1.0, 0.35, 0.35, 1)
                else:
                    b.color = (0.85, 0.85, 0.85, 1)
                b.background_color = (0.10, 0.10, 0.10, 1)
                b.bind(on_release=lambda inst, e=(name, delta, duration, cat):
                       self._add(e))
                self.cont.add_widget(b)

            for cat, items in EVENTS.items():
                hdr = Label(text="— {} —".format(cat),
                            size_hint_y=None, height=dp(30))
                hdr.color = (0.8, 0.9, 1.0, 1)
                self.cont.add_widget(hdr)

                for name, val in items:
                    if val > 0:
                        col = (0.35, 1.0, 0.35, 1)
                    elif val < 0:
                        col = (1.0, 0.35, 0.35, 1)
                    else:
                        col = (0.85, 0.85, 0.85, 1)
                    b = Button(text="{}   ({:+.2f}%)".format(name, val),
                               size_hint_y=None, height=dp(42))
                    b.color = col
                    b.background_normal = ""
                    b.background_color = (0.10, 0.10, 0.10, 1)
                    b.bind(on_release=lambda inst, v=val: self._apply(v))
                    self.cont.add_widget(b)
        except Exception:
            log_error()

    def _back(self, *a):
        if self.currency:
            self.app.open_detail(self.currency)
        else:
            self.app.sm.current = "main"

    def _apply(self, val):
        try:
            c = self.currency
            if not c:
                return
            c.rate = max(0.01, c.rate * (1.0 + val / 100.0))
            c.history.append(c.rate)
            self.app.state.turn += 1
            self.app.state.save()
            self.app.open_detail(c)
        except Exception:
            log_error()

    def _add(self, ev):
        try:
            c = self.currency
            if not c:
                return
            name, delta, duration, _cat = ev
            c.effects.append({
                "name": name,
                "delta": float(delta),
                "remaining": int(duration),
            })
            self.app.state.save()
            self._rebuild()
        except Exception:
            log_error()

    def _remove(self, idx):
        try:
            c = self.currency
            if not c:
                return
            if 0 <= idx < len(c.effects):
                c.effects.pop(idx)
                self.app.state.save()
            self._rebuild()
        except Exception:
            log_error()


# ---------- ЭКРАН СБРОСА ----------
class ResetScreen(Screen):
    def __init__(self, app, **kwargs):
        super().__init__(**kwargs)
        self.app = app
        self.stage = 0
        self._build()

    def _build(self):
        root = BoxLayout(orientation="vertical", padding=dp(20), spacing=dp(20))

        self.lbl = Label(text="", size_hint_y=None, height=dp(80))
        self.lbl.font_size = dp(22)
        self.lbl.color = (1, 1, 1, 1)
        root.add_widget(self.lbl)

        self.btn1 = Button(size_hint_y=None, height=dp(60))
        self.btn1.bind(on_release=self._action)
        root.add_widget(self.btn1)

        cancel = Button(text="Отмена", size_hint_y=None, height=dp(60))
        cancel.bind(on_release=lambda *a: self.app.go_main())
        root.add_widget(cancel)

        root.add_widget(Widget())
        self.add_widget(root)

    def on_enter(self):
        self.stage = 0
        self._refresh()

    def _refresh(self):
        if self.stage == 0:
            self.lbl.text = "Вы уверены, что хотите сбросить все данные?"
            self.btn1.text = "Да, продолжить"
            self.btn1.color = (1, 1, 1, 1)
        else:
            self.lbl.text = ("ВНИМАНИЕ!\nЭто удалит ВСЕ валюты и историю.\n"
                             "Точно сбросить?")
            self.btn1.text = "ДА, СБРОСИТЬ ВСЁ"
            self.btn1.color = (1, 0.3, 0.3, 1)

    def _action(self, *a):
        if self.stage == 0:
            self.stage = 1
            self._refresh()
        else:
            try:
                self.app.reset_all()
            except Exception:
                log_error()
            self.app.go_main()


# ---------- ПРИЛОЖЕНИЕ ----------
class CurrencyApp(App):
    def build(self):
        Window.clearcolor = (0, 0, 0, 1)
        try:
            self.state = GameState()
            self.state.load()  # без дефолтных валют

            self.sm = ScreenManager()
            self.main_screen = MainScreen(self, name="main")
            self.create_screen = CreateScreen(self, name="create")
            self.merge_screen = MergeScreen(self, name="merge")
            self.detail_screen = DetailScreen(self, name="detail")
            self.events_screen = EventsScreen(self, name="events")
            self.reset_screen = ResetScreen(self, name="reset")
            for s in (self.main_screen, self.create_screen, self.merge_screen,
                      self.detail_screen, self.events_screen, self.reset_screen):
                self.sm.add_widget(s)
            self.sm.current = "main"
            return self.sm
        except Exception:
            log_error()
            tb = traceback.format_exc()
            box = BoxLayout(orientation="vertical", padding=dp(8))
            title = Label(text="ОШИБКА при запуске:", size_hint_y=None, height=dp(40))
            title.color = (1, 0.3, 0.3, 1)
            box.add_widget(title)
            sv = ScrollView()
            lbl = Label(text=tb[-3000:])
            lbl.font_size = dp(10)
            lbl.color = (1, 0.8, 0.8, 1)
            lbl.size_hint_y = None
            lbl.bind(width=lambda inst, w: setattr(inst, "text_size", (w, None)))
            lbl.bind(texture_size=lambda inst, ts: setattr(inst, "height", ts[1]))
            sv.add_widget(lbl)
            box.add_widget(sv)
            return box

    def go_main(self):
        self.sm.current = "main"

    def open_detail(self, c):
        self.detail_screen.set_currency(c)
        self.sm.current = "detail"

    def open_events(self, c):
        self.events_screen.set_currency(c)
        self.sm.current = "events"

    def reset_all(self):
        self.state.currencies = []
        self.state.turn = 0
        self.state.save()


if __name__ == "__main__":
    try:
        CurrencyApp().run()
    except Exception:
        log_error()
        traceback.print_exc()
