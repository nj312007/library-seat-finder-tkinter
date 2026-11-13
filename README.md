import tkinter as tk
from tkinter import ttk
import random
import re
import time

# ------------------------------------------------------------
# LIBRARY SEAT FINDER — Tkinter desktop app (800 seats, dark theme)
# Run: python library_seat_finder.py
# ------------------------------------------------------------

# ---- Dark theme palette ----
BG = "#1A110B"
SIDEBAR = "#120B07"
PANEL = "#26180F"
FIELD = "#17100A"
BORDER = "#3D2A1C"
TEXT = "#F1E6D6"
MUTED = "#B9A590"
GOLD = "#D9A233"
GOLD_DARK = "#B8861F"
ON_GOLD = "#1B1109"
BTN2 = "#3A271A"
BTN2_HOVER = "#4B3322"
DISABLED_BG = "#2A1D14"
DISABLED_FG = "#6F6050"
GREEN = "#3BA76B"
RED = "#E36D75"
ORANGE = "#F2A65A"
PURPLE = "#9C8CE8"
GREY = "#7C6F64"

# ---- Seat map colours ----
MAP_BG = "#1F140D"
MAP_TABLE = "#4A2F22"
MAP_TITLE = "#F3C77A"
BOLT = "#F28C38"
STATE_COLOR = {"Available": GREEN, "Reserved": ORANGE, "Occupied": RED, "Out of service": GREY}

RESERVE_MINUTES = 15
MAX_SEATS_PER_REG = 6      # one registration number can hold at most this many seats

# ------------------------------------------------------------
# SEATING INVENTORY  (total = 800 seats)
# tables x per_table = seats in that zone.
# For single seats, per_table = 1 and "tables" = number of seats.
# "code" is used to build seat IDs such as G-6T-20-S3
# ------------------------------------------------------------
ZONES = [
    # Ground Floor = 300 seats
    {"id": "G-06",  "code": "G-6T",  "floor": "Ground Floor", "name": "6-Seater Tables",             "tables": 20,  "per_table": 6, "charging": False, "kind": "Group seating"},
    {"id": "G-04C", "code": "G-4T",  "floor": "Ground Floor", "name": "4-Seater Tables",             "tables": 25,  "per_table": 4, "charging": True,  "kind": "Group seating"},
    {"id": "G-S1",  "code": "G-1S",  "floor": "Ground Floor", "name": "Single Seats",                "tables": 80,  "per_table": 1, "charging": False, "kind": "Individual"},
    # 1st Floor = 240 seats
    {"id": "F1-06", "code": "F1-6T", "floor": "1st Floor",    "name": "6-Seater Tables",             "tables": 20,  "per_table": 6, "charging": False, "kind": "Group seating"},
    {"id": "F1-04", "code": "F1-4T", "floor": "1st Floor",    "name": "4-Seater Tables",             "tables": 10,  "per_table": 4, "charging": False, "kind": "Group seating"},
    {"id": "F1-02", "code": "F1-2T", "floor": "1st Floor",    "name": "2-Seater Tables",             "tables": 20,  "per_table": 2, "charging": False, "kind": "Group seating"},
    {"id": "F1-SC", "code": "F1-1S", "floor": "1st Floor",    "name": "Single Seats",                "tables": 40,  "per_table": 1, "charging": True,  "kind": "Individual"},
    # 2nd Floor = 260 seats
    {"id": "F2-06", "code": "F2-6T", "floor": "2nd Floor",    "name": "6-Seater Tables",             "tables": 20,  "per_table": 6, "charging": False, "kind": "Group seating"},
    {"id": "F2-EX", "code": "F2-EX", "floor": "2nd Floor",    "name": "Competitive Exam Study Hall", "tables": 140, "per_table": 1, "charging": False, "kind": "Quiet study"},
]
FLOORS = ["Ground Floor", "1st Floor", "2nd Floor", "3rd Floor"]
ZONE_BY_ID = {z["id"]: z for z in ZONES}
TABLE_LAYOUT = {6: (3, 2), 4: (2, 2), 2: (2, 1), 1: (1, 1)}   # (columns, rows)


def zone_seats(z):
    return z["tables"] * z["per_table"]


TOTAL_SEATS = sum(zone_seats(z) for z in ZONES)
assert TOTAL_SEATS == 800, f"Library should have 800 seats, found {TOTAL_SEATS}"


def all_seat_keys():
    """Seat key = (zone_id, table_number, seat_number)."""
    for z in ZONES:
        for t in range(1, z["tables"] + 1):
            for s in range(1, z["per_table"] + 1):
                yield (z["id"], t, s)


def seat_code(key):
    z = ZONE_BY_ID[key[0]]
    if z["per_table"] == 1:
        return f'{z["code"]}-{key[1]}'
    return f'{z["code"]}-{key[1]}-S{key[2]}'


def seat_place(key):
    z = ZONE_BY_ID[key[0]]
    return f'{z["floor"]} · {z["name"]}'


def clock(ts):
    return time.strftime("%H:%M", time.localtime(ts))


class SeatFinder:
    def __init__(self, root):
        self.root = root
        self.root.title("Libra — Library Seat Finder")
        self.root.geometry("1180x760")
        self.root.minsize(980, 650)
        self.root.configure(bg=BG)
        self.status = {k: "Available" for k in all_seat_keys()}
        self.holders = {}               # seat key -> {"reg","name","state","expires"}
        self.lookup_reg = ""
        self.selected_floor = "Ground Floor"
        self.map_canvas = None
        self.map_results = None
        self.res_results = None
        self.page = "Dashboard"
        self._style()
        self._shell()
        self.show_page("Dashboard")
        self.root.after(10000, self._tick)

    # ---------------------------------------------------------- setup
    def _style(self):
        style = ttk.Style()
        style.theme_use("clam")
        style.configure("TCombobox", padding=7, fieldbackground=FIELD, background=BTN2,
                        foreground=TEXT, arrowcolor=GOLD, bordercolor=BORDER,
                        lightcolor=BORDER, darkcolor=BORDER,
                        selectbackground=FIELD, selectforeground=TEXT)
        style.map("TCombobox",
                  fieldbackground=[("readonly", FIELD)],
                  foreground=[("readonly", TEXT)],
                  selectbackground=[("readonly", FIELD)],
                  selectforeground=[("readonly", TEXT)])
        self.root.option_add("*TCombobox*Listbox.background", FIELD)
        self.root.option_add("*TCombobox*Listbox.foreground", TEXT)
        self.root.option_add("*TCombobox*Listbox.selectBackground", GOLD)
        self.root.option_add("*TCombobox*Listbox.selectForeground", ON_GOLD)
        style.configure("TCheckbutton", background=PANEL, foreground=TEXT, font=("Segoe UI", 10))
        style.map("TCheckbutton", background=[("active", PANEL)],
                  foreground=[("active", TEXT)],
                  indicatorcolor=[("selected", GOLD), ("!selected", FIELD)])
        style.configure("Vertical.TScrollbar", background=MAP_TABLE, troughcolor=MAP_BG,
                        bordercolor=MAP_BG, arrowcolor=GOLD, lightcolor=MAP_TABLE, darkcolor=MAP_TABLE)

    def _shell(self):
        self.sidebar = tk.Frame(self.root, bg=SIDEBAR, width=225)
        self.sidebar.pack(side="left", fill="y")
        self.sidebar.pack_propagate(False)

        brand = tk.Frame(self.sidebar, bg=SIDEBAR)
        brand.pack(fill="x", padx=22, pady=(25, 34))
        tk.Label(brand, text="✦  LIBRA", bg=SIDEBAR, fg=GOLD,
                 font=("Georgia", 20, "bold")).pack(anchor="w")
        tk.Label(brand, text="LIBRARY SEAT FINDER", bg=SIDEBAR, fg=MUTED,
                 font=("Segoe UI", 8, "bold")).pack(anchor="w", pady=(4, 0))

        self.nav_buttons = {}
        for label, icon in [("Dashboard", "⌂"), ("Find a Seat", "⌕"), ("Seat Map", "▦"),
                            ("My Reservation", "▣"), ("Library Guide", "ⓘ")]:
            b = tk.Button(self.sidebar, text=f"   {icon}    {label}", anchor="w",
                          command=lambda p=label: self.show_page(p), relief="flat",
                          bd=0, bg=SIDEBAR, fg=MUTED, activebackground=PANEL,
                          activeforeground=GOLD, font=("Segoe UI", 11), padx=14, pady=13,
                          cursor="hand2")
            b.pack(fill="x", padx=12, pady=2)
            self.nav_buttons[label] = b

        tk.Frame(self.sidebar, bg=BORDER, height=1).pack(fill="x", padx=20, pady=(25, 16))
        tk.Label(self.sidebar, text="STUDENT ACCESS", bg=SIDEBAR, fg=MUTED,
                 font=("Segoe UI", 8, "bold")).pack(anchor="w", padx=25)
        tk.Label(self.sidebar, text="3rd floor is reserved\nfor library staff.",
                 bg=SIDEBAR, fg=TEXT, justify="left",
                 font=("Segoe UI", 9), pady=9).pack(anchor="w", padx=25)

        self.main = tk.Frame(self.root, bg=BG)
        self.main.pack(side="right", fill="both", expand=True)
        self.header = tk.Frame(self.main, bg=BG)
        self.header.pack(fill="x", padx=32, pady=(24, 12))
        self.page_title = tk.Label(self.header, text="", bg=BG, fg=MAP_TITLE,
                                   font=("Georgia", 22, "bold"))
        self.page_title.pack(anchor="w")
        self.page_subtitle = tk.Label(self.header, text="", bg=BG, fg=MUTED,
                                      font=("Segoe UI", 10))
        self.page_subtitle.pack(anchor="w", pady=(4, 0))
        self.content = tk.Frame(self.main, bg=BG)
        self.content.pack(fill="both", expand=True, padx=32, pady=(8, 28))

    def clear_content(self):
        self.map_canvas = None
        self.map_results = None
        self.res_results = None
        for w in self.content.winfo_children():
            w.destroy()

    # ---------------------------------------------------------- widgets
    def card(self, parent, padx=18, pady=16):
        return tk.Frame(parent, bg=PANEL, highlightbackground=BORDER,
                        highlightthickness=1, padx=padx, pady=pady)

    def style_btn(self, b, enabled=True, primary=True):
        if enabled:
            b.configure(state="normal", cursor="hand2",
                        bg=GOLD if primary else BTN2,
                        fg=ON_GOLD if primary else TEXT,
                        activebackground=GOLD_DARK if primary else BTN2_HOVER,
                        activeforeground=ON_GOLD if primary else TEXT)
        else:
            b.configure(state="disabled", cursor="", bg=DISABLED_BG, fg=DISABLED_FG,
                        disabledforeground=DISABLED_FG)

    def button(self, parent, text, command, primary=True, width=None):
        b = tk.Button(parent, text=text, command=command, relief="flat", bd=0,
                      padx=16, pady=10, font=("Segoe UI", 9, "bold"), width=width,
                      disabledforeground=DISABLED_FG)
        self.style_btn(b, True, primary)
        return b

    def field(self, parent, var):
        return tk.Entry(parent, textvariable=var, bg=FIELD, fg=TEXT, insertbackground=TEXT,
                        relief="flat", font=("Segoe UI", 11), highlightthickness=1,
                        highlightbackground=BORDER, highlightcolor=GOLD)

    def modal(self, title):
        win = tk.Toplevel(self.root)
        win.title(title)
        win.configure(bg=BG)
        win.transient(self.root)
        win.resizable(False, False)
        return win

    def present(self, win):
        win.update_idletasks()
        x = self.root.winfo_rootx() + (self.root.winfo_width() - win.winfo_width()) // 2
        y = self.root.winfo_rooty() + (self.root.winfo_height() - win.winfo_height()) // 3
        win.geometry(f"+{max(x, 0)}+{max(y, 0)}")
        try:
            win.grab_set()
        except tk.TclError:
            pass
        win.focus_set()

    def notice(self, title, message):
        win = self.modal(title)
        box = self.card(win, 24, 20)
        box.pack(padx=18, pady=18)
        tk.Label(box, text=title, bg=PANEL, fg=MAP_TITLE, font=("Georgia", 15, "bold")).pack(anchor="w")
        tk.Label(box, text=message, bg=PANEL, fg=TEXT, font=("Segoe UI", 10),
                 justify="left", wraplength=340).pack(anchor="w", pady=(8, 16))
        self.button(box, "OK", win.destroy).pack(anchor="w")
        self.present(win)
        win.wait_window()

    # ---------------------------------------------------------- data helpers
    def count(self, state, zone_id=None):
        return sum(1 for k, v in self.status.items()
                   if v == state and (zone_id is None or k[0] == zone_id))

    def floor_total(self, floor):
        return sum(zone_seats(z) for z in ZONES if z["floor"] == floor)

    def floor_desc(self, floor):
        if floor == "3rd Floor":
            return "Staff area — student access restricted"
        parts = [f'{zone_seats(z)} {z["name"].lower()}' + (" (charging)" if z["charging"] else "")
                 for z in ZONES if z["floor"] == floor]
        return "  ·  ".join(parts) + f"   —   {self.floor_total(floor)} seats"

    def holding_count(self, reg):
        return sum(1 for h in self.holders.values() if h["reg"] == reg)

    @staticmethod
    def norm_reg(text):
        return text.strip().upper().replace(" ", "")

    def clean_inputs(self, reg_var, name_var):
        reg = self.norm_reg(reg_var.get())
        name = " ".join(name_var.get().split())
        if not re.fullmatch(r"[A-Z0-9]{5,20}", reg):
            return None, None, "Enter a valid registration number (letters and digits only)."
        if len(name) < 2:
            return None, None, "Enter your name."
        return reg, name, None

    def do_reserve(self, keys, reg, name):
        exp = time.time() + RESERVE_MINUTES * 60
        for k in keys:
            self.status[k] = "Reserved"
            self.holders[k] = {"reg": reg, "name": name, "state": "Reserved", "expires": exp}

    def do_checkin(self, key, reg, name):
        self.status[key] = "Occupied"
        self.holders[key] = {"reg": reg, "name": name, "state": "Checked in", "expires": None}

    def do_release(self, key):
        self.status[key] = "Available"
        self.holders.pop(key, None)

    def refresh_views(self):
        """Redraw whatever is on screen after a seat changed."""
        try:
            if self.page == "Seat Map" and self.map_results is not None:
                self.render_floor(keep_scroll=True)
            elif self.page == "Dashboard":
                self.show_page("Dashboard")
            elif self.page == "My Reservation" and self.res_results is not None:
                self.run_lookup()
            elif self.page == "Find a Seat" and self.res_results is None and hasattr(self, "results"):
                if self.results.winfo_exists() and self.results.winfo_children():
                    self.search_seats()
        except tk.TclError:
            pass

    def _tick(self):
        now = time.time()
        expired = [k for k, h in self.holders.items()
                   if h["state"] == "Reserved" and h["expires"] and h["expires"] <= now]
        for k in expired:
            self.do_release(k)
        if expired:
            self.refresh_views()
        self.root.after(10000, self._tick)

    # ---------------------------------------------------------- pages
    def show_page(self, page):
        self.page = page
        for name, b in self.nav_buttons.items():
            b.configure(bg=PANEL if name == page else SIDEBAR,
                        fg=GOLD if name == page else MUTED)
        self.clear_content()
        titles = {
            "Dashboard": ("Welcome to your library", f"{TOTAL_SEATS} seats across 3 student floors. Choose a floor or let Smart Find do it for you."),
            "Find a Seat": ("Smart Find", "Tell us what you need and discover matching seating areas."),
            "Seat Map": ("Library seat map", "Click any seat to reserve, check in or release it with your registration number."),
            "My Reservation": ("My Reservation", "Look up the seats held under your registration number."),
            "Library Guide": ("Library Guide", "A quick overview of the library's floor arrangement.")
        }
        self.page_title.configure(text=titles[page][0])
        self.page_subtitle.configure(text=titles[page][1])
        {"Dashboard": self.dashboard, "Find a Seat": self.find_page,
         "Seat Map": self.map_page, "My Reservation": self.reservation_page,
         "Library Guide": self.guide_page}[page]()

    def stats(self, parent):
        items = [("AVAILABLE", self.count("Available"), GREEN),
                 ("OCCUPIED", self.count("Occupied"), RED),
                 ("RESERVED", self.count("Reserved"), ORANGE),
                 ("OUT OF SERVICE", self.count("Out of service"), GREY),
                 ("TOTAL SEATS", TOTAL_SEATS, PURPLE)]
        row = tk.Frame(parent, bg=BG)
        row.pack(fill="x", pady=(0, 18))
        for title, value, color in items:
            c = self.card(row, 15, 13)
            c.pack(side="left", fill="both", expand=True, padx=(0, 12))
            tk.Label(c, text=title, bg=PANEL, fg=MUTED,
                     font=("Segoe UI", 8, "bold")).pack(anchor="w")
            tk.Label(c, text=str(value), bg=PANEL, fg=color,
                     font=("Segoe UI", 22, "bold")).pack(anchor="w", pady=(4, 0))

    def dashboard(self):
        self.stats(self.content)
        hero = tk.Frame(self.content, bg="#2E1E13", padx=25, pady=22,
                        highlightbackground=BORDER, highlightthickness=1)
        hero.pack(fill="x", pady=(0, 18))
        tk.Label(hero, text="Your next productive hour starts here.",
                 bg="#2E1E13", fg=MAP_TITLE, font=("Georgia", 17, "bold")).pack(anchor="w")
        tk.Label(hero, text="Find a spot that fits your group, charging and quiet-study needs.",
                 bg="#2E1E13", fg=MUTED, font=("Segoe UI", 10)).pack(anchor="w", pady=(6, 15))
        btns = tk.Frame(hero, bg="#2E1E13")
        btns.pack(anchor="w")
        self.button(btns, "Find my seat  →", lambda: self.show_page("Find a Seat")).pack(side="left", padx=(0, 10))
        self.button(btns, "Open seat map  ▦", lambda: self.show_page("Seat Map"), primary=False).pack(side="left")

        tk.Label(self.content, text="BROWSE BY FLOOR", bg=BG, fg=MUTED,
                 font=("Segoe UI", 9, "bold")).pack(anchor="w", pady=(2, 10))
        grid = tk.Frame(self.content, bg=BG)
        grid.pack(fill="x")
        icons = {"Ground Floor": "G", "1st Floor": "1", "2nd Floor": "2", "3rd Floor": "3"}
        for i, floor in enumerate(FLOORS):
            c = self.card(grid, 17, 15)
            c.grid(row=i // 2, column=i % 2, sticky="nsew", padx=(0, 12), pady=(0, 12))
            tk.Label(c, text=f"FLOOR {icons[floor]}", bg=PANEL, fg=GOLD,
                     font=("Segoe UI", 8, "bold")).pack(anchor="w")
            tk.Label(c, text=floor, bg=PANEL, fg=TEXT,
                     font=("Segoe UI", 14, "bold")).pack(anchor="w", pady=(5, 3))
            tk.Label(c, text=self.floor_desc(floor), bg=PANEL, fg=MUTED, wraplength=360,
                     justify="left", font=("Segoe UI", 9)).pack(anchor="w", pady=(0, 12))
            self.button(c, "Explore floor  →",
                        lambda f=floor: self.open_floor(f), primary=False).pack(anchor="w")
        grid.columnconfigure(0, weight=1)
        grid.columnconfigure(1, weight=1)

        foot = tk.Frame(self.content, bg=BG)
        foot.pack(fill="x", pady=(8, 0))
        tk.Label(foot, text="DEMO MODE  ·  Seat availability is sample data and is not connected to live sensors.",
                 bg=BG, fg=MUTED, font=("Segoe UI", 8)).pack(anchor="w")
        self.button(foot, "Simulate busy hours", self.simulate_busy, primary=False).pack(anchor="e", pady=(8, 0))

    def open_floor(self, floor):
        self.selected_floor = floor
        self.show_page("Seat Map")

    # ---------------------------------------------------------- smart find
    def find_page(self):
        card = self.card(self.content, 22, 20)
        card.pack(fill="x", anchor="n")
        tk.Label(card, text="What does your study session look like?",
                 bg=PANEL, fg=TEXT, font=("Segoe UI", 14, "bold")).pack(anchor="w")
        tk.Label(card, text="Choose your preferences below. Matching areas will appear underneath.",
                 bg=PANEL, fg=MUTED, font=("Segoe UI", 9)).pack(anchor="w", pady=(4, 18))
        form = tk.Frame(card, bg=PANEL)
        form.pack(fill="x")
        tk.Label(form, text="GROUP SIZE", bg=PANEL, fg=MUTED,
                 font=("Segoe UI", 8, "bold")).grid(row=0, column=0, sticky="w")
        self.group_var = tk.StringVar(value="1")
        ttk.Combobox(form, textvariable=self.group_var, values=["1", "2", "3", "4", "5", "6"],
                     state="readonly", width=18).grid(row=1, column=0, sticky="w", pady=(6, 15), padx=(0, 25))
        tk.Label(form, text="PREFERRED FLOOR", bg=PANEL, fg=MUTED,
                 font=("Segoe UI", 8, "bold")).grid(row=0, column=1, sticky="w")
        self.floor_var = tk.StringVar(value="Any floor")
        ttk.Combobox(form, textvariable=self.floor_var,
                     values=["Any floor", "Ground Floor", "1st Floor", "2nd Floor"],
                     state="readonly", width=22).grid(row=1, column=1, sticky="w", pady=(6, 15))
        self.charge_var = tk.BooleanVar()
        self.quiet_var = tk.BooleanVar()
        ttk.Checkbutton(form, text="I need a charging point", variable=self.charge_var).grid(row=2, column=0, sticky="w")
        ttk.Checkbutton(form, text="Prefer a quiet study area", variable=self.quiet_var).grid(row=2, column=1, sticky="w")
        self.button(card, "Find matching seats  →", self.search_seats).pack(anchor="w", pady=(20, 0))
        self.results = tk.Frame(self.content, bg=BG)
        self.results.pack(fill="both", expand=True, pady=(18, 0))
        tk.Label(self.results, text="Your matches will show here.",
                 bg=BG, fg=MUTED, font=("Segoe UI", 10)).pack(anchor="w")

    def best_table(self, z, size):
        """Return `size` free seat keys on one table of zone z, or None."""
        if z["per_table"] == 1:
            if size != 1:
                return None
            for t in range(1, z["tables"] + 1):
                if self.status[(z["id"], t, 1)] == "Available":
                    return [(z["id"], t, 1)]
            return None
        for t in range(1, z["tables"] + 1):
            free = [(z["id"], t, s) for s in range(1, z["per_table"] + 1)
                    if self.status[(z["id"], t, s)] == "Available"]
            if len(free) >= size:
                return free[:size]
        return None

    def search_seats(self):
        for w in self.results.winfo_children():
            w.destroy()
        size = int(self.group_var.get())
        floor = self.floor_var.get()
        matches = []
        for z in ZONES:
            if floor != "Any floor" and z["floor"] != floor:
                continue
            if z["per_table"] < size:
                continue
            if self.charge_var.get() and not z["charging"]:
                continue
            if self.quiet_var.get() and z["kind"] != "Quiet study":
                continue
            seats = self.best_table(z, size)
            if seats:
                matches.append((z, seats))
        matches.sort(key=lambda m: m[0]["per_table"])
        tk.Label(self.results, text=f"{len(matches)} MATCHING AREA(S)", bg=BG, fg=MUTED,
                 font=("Segoe UI", 9, "bold")).pack(anchor="w", pady=(0, 8))
        if not matches:
            tk.Label(self.results, text="No matching area found. Try changing your filters.",
                     bg=BG, fg=TEXT, font=("Segoe UI", 10)).pack(anchor="w")
        for z, seats in matches[:6]:
            c = self.card(self.results, 15, 13)
            c.pack(fill="x", pady=(0, 8))
            head = tk.Frame(c, bg=PANEL)
            head.pack(fill="x")
            tk.Label(head, text=z["name"], bg=PANEL, fg=TEXT,
                     font=("Segoe UI", 12, "bold")).pack(side="left")
            tk.Label(head, text=f'{self.count("Available", zone_id=z["id"])} FREE', bg=GREEN, fg="white",
                     font=("Segoe UI", 8, "bold"), padx=9, pady=4).pack(side="right")
            details = f'{z["floor"]}  ·  {z["per_table"]} seat' + ("s" if z["per_table"] > 1 else "") + " per table"
            if z["charging"]:
                details += "  ·  ⚡ Charging"
            tk.Label(c, text=details, bg=PANEL, fg=MUTED, font=("Segoe UI", 9)).pack(anchor="w", pady=(8, 4))
            tk.Label(c, text="Suggested: " + ", ".join(seat_code(k) for k in seats), bg=PANEL, fg=MAP_TITLE,
                     font=("Segoe UI", 9, "bold")).pack(anchor="w", pady=(0, 10))
            row = tk.Frame(c, bg=PANEL)
            row.pack(fill="x")
            self.button(row, "Show on map", lambda f=z["floor"]: self.open_floor(f), primary=False).pack(side="left")
            self.button(row, f"Reserve {size} seat" + ("s" if size > 1 else "") + f" · {RESERVE_MINUTES} min",
                        lambda s=seats: self.group_dialog(s)).pack(side="right")

    def group_dialog(self, keys):
        win = self.modal("Reserve seats")
        box = self.card(win, 26, 22)
        box.pack(padx=16, pady=16)
        n = len(keys)
        tk.Label(box, text=f"Reserve {n} seat" + ("s" if n > 1 else ""), bg=PANEL, fg=MAP_TITLE,
                 font=("Georgia", 16, "bold")).pack(anchor="w")
        tk.Label(box, text=", ".join(seat_code(k) for k in keys), bg=PANEL, fg=MUTED,
                 font=("Segoe UI", 9), wraplength=360, justify="left").pack(anchor="w", pady=(4, 14))
        reg_var, name_var = tk.StringVar(), tk.StringVar()
        tk.Label(box, text="Registration number", bg=PANEL, fg=TEXT, font=("Segoe UI", 10)).pack(anchor="w")
        e1 = self.field(box, reg_var)
        e1.pack(fill="x", ipady=8, pady=(4, 12))
        tk.Label(box, text="Name", bg=PANEL, fg=TEXT, font=("Segoe UI", 10)).pack(anchor="w")
        self.field(box, name_var).pack(fill="x", ipady=8, pady=(4, 8))
        msg = tk.Label(box, text="", bg=PANEL, fg=RED, font=("Segoe UI", 9), wraplength=360, justify="left")
        msg.pack(anchor="w", pady=(0, 8))

        def confirm():
            reg, name, err = self.clean_inputs(reg_var, name_var)
            if err:
                msg.configure(text=err)
                return
            if any(self.status[k] != "Available" for k in keys):
                msg.configure(text="One of these seats was just taken. Close and search again.")
                return
            if self.holding_count(reg) + n > MAX_SEATS_PER_REG:
                msg.configure(text=f"One registration number can hold at most {MAX_SEATS_PER_REG} seats.")
                return
            self.do_reserve(keys, reg, name)
            win.destroy()
            self.refresh_views()
            self.notice("Seats reserved",
                        f"{n} seat{'s' if n > 1 else ''} reserved for {RESERVE_MINUTES} minutes.\n"
                        f"Check in before {clock(time.time() + RESERVE_MINUTES * 60)}.")

        row = tk.Frame(box, bg=PANEL)
        row.pack(fill="x")
        self.button(row, f"Reserve {RESERVE_MINUTES} min", confirm).pack(side="left", padx=(0, 8))
        self.button(row, "Cancel", win.destroy, primary=False).pack(side="left")
        self.present(win)
        e1.focus_set()

    # ---------------------------------------------------------- seat map
    def map_page(self):
        top = tk.Frame(self.content, bg=BG)
        top.pack(fill="x", pady=(0, 10))
        self.floor_buttons = {}
        for f in FLOORS:
            b = self.button(top, f, lambda x=f: self.set_floor(x), primary=(f == self.selected_floor))
            b.pack(side="left", padx=(0, 8))
            self.floor_buttons[f] = b

        legend = tk.Frame(top, bg=BG)
        legend.pack(side="right")
        for text, color in [("Free", GREEN), ("Reserved", ORANGE), ("Occupied", RED), ("Out of service", GREY)]:
            tk.Label(legend, text="■", bg=BG, fg=color, font=("Segoe UI", 13)).pack(side="left")
            tk.Label(legend, text=text + "  ", bg=BG, fg=MUTED, font=("Segoe UI", 9)).pack(side="left")
        tk.Label(legend, text="⚡ Charging", bg=BG, fg=BOLT, font=("Segoe UI", 9, "bold")).pack(side="left")

        self.map_results = tk.Frame(self.content, bg=BG)
        self.map_results.pack(fill="both", expand=True)
        self.render_floor()

    def set_floor(self, floor):
        self.selected_floor = floor
        for f, b in self.floor_buttons.items():
            self.style_btn(b, True, primary=(f == floor))
        self.render_floor()

    def rounded(self, c, x1, y1, x2, y2, r, **kw):
        pts = [x1 + r, y1, x2 - r, y1, x2, y1, x2, y1 + r, x2, y2 - r, x2, y2,
               x2 - r, y2, x1 + r, y2, x1, y2, x1, y2 - r, x1, y1 + r, x1, y1]
        return c.create_polygon(pts, smooth=True, **kw)

    def render_floor(self, keep_scroll=False):
        pos = 0
        if keep_scroll and self.map_canvas is not None:
            pos = self.map_canvas.yview()[0]
        for w in self.map_results.winfo_children():
            w.destroy()
        self.map_canvas = None
        f = self.selected_floor

        if f == "3rd Floor":
            c = self.card(self.map_results, 25, 25)
            c.pack(fill="x")
            tk.Label(c, text="STAFF ONLY", bg=PANEL, fg=RED,
                     font=("Segoe UI", 15, "bold")).pack(anchor="w")
            tk.Label(c, text="The 3rd floor is reserved for library staff. Student seating is not available here.",
                     bg=PANEL, fg=MUTED, font=("Segoe UI", 10)).pack(anchor="w", pady=(8, 0))
            return

        wrap = tk.Frame(self.map_results, bg=MAP_BG, highlightbackground=BORDER, highlightthickness=1)
        wrap.pack(fill="both", expand=True)
        canvas = tk.Canvas(wrap, bg=MAP_BG, highlightthickness=0)
        sb = ttk.Scrollbar(wrap, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=sb.set)
        sb.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)
        self.map_canvas = canvas
        self.root.update_idletasks()
        W = max(620, canvas.winfo_width() - 10)

        SEAT, GAP, PAD, TGAP = 26, 4, 6, 10
        y = 12
        for z in [z for z in ZONES if z["floor"] == f]:
            free = self.count("Available", zone_id=z["id"])
            total = zone_seats(z)
            title = z["name"] + ("  ⚡" if z["charging"] else "")
            tid = canvas.create_text(12, y, text=title, anchor="nw", fill=MAP_TITLE,
                                     font=("Georgia", 15, "bold"))
            bx = canvas.bbox(tid)
            canvas.create_text(bx[2] + 14, y + 6, text=f"{free} of {total} free", anchor="nw",
                               fill=MUTED, font=("Segoe UI", 9))
            y += 36

            cols, rows = TABLE_LAYOUT[z["per_table"]]
            single = z["per_table"] == 1
            if single:
                tw = th = SEAT
                step_x = SEAT + 8
            else:
                tw = cols * SEAT + (cols - 1) * GAP + 2 * PAD
                th = rows * SEAT + (rows - 1) * GAP + 2 * PAD
                step_x = tw + TGAP
            step_y = th + TGAP

            x = 12
            row_y = y
            for t in range(1, z["tables"] + 1):
                if x + tw > W - 6:
                    x = 12
                    row_y += step_y
                if not single:
                    self.rounded(canvas, x, row_y, x + tw, row_y + th, 8, fill=MAP_TABLE, outline="")
                for s in range(1, z["per_table"] + 1):
                    r_i, c_i = divmod(s - 1, cols)
                    sx = x + (0 if single else PAD + c_i * (SEAT + GAP))
                    sy = row_y + (0 if single else PAD + r_i * (SEAT + GAP))
                    key = (z["id"], t, s)
                    st = self.status[key]
                    item = self.rounded(canvas, sx, sy, sx + SEAT, sy + SEAT, 6, fill=STATE_COLOR[st], outline="")
                    items = [item]
                    if z["charging"]:
                        items.append(canvas.create_text(sx + SEAT / 2, sy + SEAT / 2, text="⚡",
                                                        fill=BOLT if st == "Available" else TEXT,
                                                        font=("Segoe UI", 9)))
                    for it in items:
                        canvas.tag_bind(it, "<Button-1>", lambda e, k=key: self.open_seat_dialog(k))
                        canvas.tag_bind(it, "<Enter>", lambda e: canvas.configure(cursor="hand2"))
                        canvas.tag_bind(it, "<Leave>", lambda e: canvas.configure(cursor=""))
                x += step_x
            y = row_y + th + 26

        canvas.configure(scrollregion=(0, 0, W, y))
        canvas.yview_moveto(pos)
        self._bind_wheel(canvas)

    def _bind_wheel(self, canvas):
        def on_wheel(e):
            try:
                if not canvas.winfo_exists():
                    return
                if getattr(e, "num", 0) == 4:
                    canvas.yview_scroll(-3, "units")
                elif getattr(e, "num", 0) == 5:
                    canvas.yview_scroll(3, "units")
                else:
                    canvas.yview_scroll(int(-e.delta / 40) or (-1 if e.delta > 0 else 1), "units")
            except tk.TclError:
                pass
        canvas.bind("<Enter>", lambda e: (self.root.bind_all("<MouseWheel>", on_wheel),
                                          self.root.bind_all("<Button-4>", on_wheel),
                                          self.root.bind_all("<Button-5>", on_wheel)))
        canvas.bind("<Leave>", lambda e: (self.root.unbind_all("<MouseWheel>"),
                                          self.root.unbind_all("<Button-4>"),
                                          self.root.unbind_all("<Button-5>")))

    # ---------------------------------------------------------- seat dialog
    def open_seat_dialog(self, key):
        z = ZONE_BY_ID[key[0]]
        win = self.modal(seat_code(key))
        box = tk.Frame(win, bg=PANEL, highlightbackground="#5A3F25", highlightthickness=1,
                       padx=24, pady=22)
        box.pack(padx=16, pady=16)

        tk.Label(box, text=seat_code(key), bg=PANEL, fg=MAP_TITLE,
                 font=("Georgia", 17, "bold")).pack(anchor="w")
        tk.Label(box, text=seat_place(key) + ("  ·  ⚡ Charging" if z["charging"] else ""),
                 bg=PANEL, fg=MUTED, font=("Segoe UI", 9)).pack(anchor="w", pady=(6, 0))
        status_lbl = tk.Label(box, text="", bg=PANEL, fg=MUTED, font=("Segoe UI", 9, "bold"))
        status_lbl.pack(anchor="w", pady=(2, 12))

        reg_var, name_var = tk.StringVar(), tk.StringVar()
        tk.Label(box, text="Registration number", bg=PANEL, fg=TEXT, font=("Segoe UI", 11)).pack(anchor="w")
        reg_entry = self.field(box, reg_var)
        reg_entry.pack(fill="x", ipady=8, pady=(5, 12))
        tk.Label(box, text="Name", bg=PANEL, fg=TEXT, font=("Segoe UI", 11)).pack(anchor="w")
        self.field(box, name_var).pack(fill="x", ipady=8, pady=(5, 10))

        msg = tk.Label(box, text="", bg=PANEL, fg=RED, font=("Segoe UI", 9),
                       wraplength=340, justify="left")
        msg.pack(anchor="w", pady=(0, 6))

        def say(text, color=RED):
            msg.configure(text=text, fg=color)

        row1 = tk.Frame(box, bg=PANEL)
        row1.pack(fill="x")
        row2 = tk.Frame(box, bg=PANEL)
        row2.pack(fill="x", pady=(8, 0))

        def perms():
            st = self.status[key]
            return {"checkin": st in ("Available", "Reserved"),
                    "reserve": st == "Available",
                    "release": st in ("Reserved", "Occupied"),
                    "service": True}

        def sync():
            st = self.status[key]
            h = self.holders.get(key)
            text = f"Status: {st}"
            if st == "Reserved" and h and h["expires"]:
                text += f"  ·  hold ends {clock(h['expires'])}"
            status_lbl.configure(text=text, fg=STATE_COLOR[st])
            p = perms()
            self.style_btn(b_checkin, p["checkin"], True)
            self.style_btn(b_reserve, p["reserve"], True)
            self.style_btn(b_release, p["release"], True)
            b_service.configure(text="Back in service" if st == "Out of service" else "Out of service")

        def guard(name):
            if not perms()[name]:
                sync()
                say("That seat's status just changed. Please check and try again.")
                return False
            return True

        def finish(text):
            self.refresh_views()
            sync()
            say(text, GREEN)

        def check_in():
            if not guard("checkin"):
                return
            reg, name, err = self.clean_inputs(reg_var, name_var)
            if err:
                return say(err)
            h = self.holders.get(key)
            if self.status[key] == "Reserved":
                if not h or h["reg"] != reg:
                    return say("This seat is reserved by another student.")
            elif self.holding_count(reg) >= MAX_SEATS_PER_REG:
                return say(f"One registration number can hold at most {MAX_SEATS_PER_REG} seats.")
            self.do_checkin(key, reg, name)
            finish("Checked in. Happy studying!")

        def reserve():
            if not guard("reserve"):
                return
            reg, name, err = self.clean_inputs(reg_var, name_var)
            if err:
                return say(err)
            if self.holding_count(reg) >= MAX_SEATS_PER_REG:
                return say(f"One registration number can hold at most {MAX_SEATS_PER_REG} seats.")
            self.do_reserve([key], reg, name)
            finish(f"Reserved for {RESERVE_MINUTES} minutes. Check in before "
                   f"{clock(time.time() + RESERVE_MINUTES * 60)}.")

        def release():
            if not guard("release"):
                return
            h = self.holders.get(key)
            if h and self.norm_reg(reg_var.get()) != h["reg"]:
                return say("Enter the registration number used for this seat to release it.")
            self.do_release(key)
            finish("Seat released.")

        def toggle_service():
            if self.status[key] == "Out of service":
                self.status[key] = "Available"
                finish("Seat is back in service.")
            else:
                self.holders.pop(key, None)
                self.status[key] = "Out of service"
                finish("Seat marked out of service.")

        b_checkin = self.button(row1, "Check in", check_in)
        b_checkin.pack(side="left", padx=(0, 8))
        b_reserve = self.button(row1, f"Reserve {RESERVE_MINUTES} min", reserve)
        b_reserve.pack(side="left", padx=(0, 8))
        b_release = self.button(row1, "Release", release)
        b_release.pack(side="left")
        b_service = self.button(row2, "Out of service", toggle_service, primary=False)
        b_service.pack(side="left", padx=(0, 8))
        self.button(row2, "Close", win.destroy, primary=False).pack(side="left")

        sync()
        self.present(win)
        reg_entry.focus_set()

    # ---------------------------------------------------------- my reservation
    def reservation_page(self):
        c = self.card(self.content, 22, 18)
        c.pack(fill="x")
        tk.Label(c, text="Find your seats", bg=PANEL, fg=TEXT, font=("Segoe UI", 14, "bold")).pack(anchor="w")
        tk.Label(c, text="Enter the registration number you used to reserve or check in.",
                 bg=PANEL, fg=MUTED, font=("Segoe UI", 9)).pack(anchor="w", pady=(4, 12))
        row = tk.Frame(c, bg=PANEL)
        row.pack(fill="x")
        self.lookup_var = tk.StringVar(value=self.lookup_reg)
        e = self.field(row, self.lookup_var)
        e.pack(side="left", fill="x", expand=True, ipady=8, padx=(0, 10))
        e.bind("<Return>", lambda ev: self.run_lookup())
        self.button(row, "Find my seats", self.run_lookup).pack(side="left")
        self.res_results = tk.Frame(self.content, bg=BG)
        self.res_results.pack(fill="both", expand=True, pady=(16, 0))
        self.run_lookup()

    def run_lookup(self):
        for w in self.res_results.winfo_children():
            w.destroy()
        reg = self.norm_reg(self.lookup_var.get())
        self.lookup_reg = reg
        if not reg:
            tk.Label(self.res_results, text="Your reserved and checked-in seats will show here.",
                     bg=BG, fg=MUTED, font=("Segoe UI", 10)).pack(anchor="w")
            return
        mine = sorted(((k, h) for k, h in self.holders.items() if h["reg"] == reg),
                      key=lambda kh: seat_code(kh[0]))
        if not mine:
            tk.Label(self.res_results, text=f"No active reservation found for {reg}.",
                     bg=BG, fg=TEXT, font=("Segoe UI", 10)).pack(anchor="w", pady=(0, 10))
            r = tk.Frame(self.res_results, bg=BG)
            r.pack(anchor="w")
            self.button(r, "Find a seat", lambda: self.show_page("Find a Seat")).pack(side="left", padx=(0, 8))
            self.button(r, "Open seat map", lambda: self.show_page("Seat Map"), primary=False).pack(side="left")
            return
        for key, h in mine:
            c = self.card(self.res_results, 22, 16)
            c.pack(fill="x", pady=(0, 10))
            head = tk.Frame(c, bg=PANEL)
            head.pack(fill="x")
            tk.Label(head, text=seat_code(key), bg=PANEL, fg=MAP_TITLE,
                     font=("Georgia", 14, "bold")).pack(side="left")
            tk.Label(head, text=h["state"].upper(), bg=ORANGE if h["state"] == "Reserved" else RED,
                     fg=ON_GOLD, font=("Segoe UI", 8, "bold"), padx=9, pady=4).pack(side="right")
            detail = f'{seat_place(key)}  ·  {h["name"]}'
            if h["state"] == "Reserved" and h["expires"]:
                detail += f'  ·  hold ends {clock(h["expires"])}'
            tk.Label(c, text=detail, bg=PANEL, fg=MUTED).pack(anchor="w", pady=(6, 12))
            r = tk.Frame(c, bg=PANEL)
            r.pack(fill="x")
            if h["state"] == "Reserved":
                self.button(r, "Check in", lambda k=key: self.list_checkin(k)).pack(side="left", padx=(0, 8))
            self.button(r, "Release seat", lambda k=key: self.list_release(k), primary=False).pack(side="left")

    def list_checkin(self, key):
        h = self.holders.get(key)
        if h:
            self.do_checkin(key, h["reg"], h["name"])
        self.refresh_views()
        self.notice("Checked in", "You are checked in. Happy studying!")

    def list_release(self, key):
        self.do_release(key)
        self.refresh_views()
        self.notice("Released", "The seat is available again.")

    # ---------------------------------------------------------- guide
    def guide_page(self):
        for floor in FLOORS:
            c = self.card(self.content, 18, 14)
            c.pack(fill="x", pady=(0, 9))
            tk.Label(c, text=floor, bg=PANEL, fg=TEXT, font=("Segoe UI", 12, "bold")).pack(anchor="w")
            tk.Label(c, text=self.floor_desc(floor), bg=PANEL, fg=MUTED, font=("Segoe UI", 9),
                     wraplength=780, justify="left").pack(anchor="w", pady=(5, 0))
        tk.Label(self.content,
                 text=f"Total student seating: {TOTAL_SEATS} seats.  Reservations hold a seat for {RESERVE_MINUTES} minutes; "
                      f"one registration number can hold up to {MAX_SEATS_PER_REG} seats.  "
                      "Note: this is a front-end demo — connect it to a database for real availability.",
                 bg=BG, fg=MUTED, font=("Segoe UI", 8), wraplength=800, justify="left").pack(anchor="w", pady=(8, 0))

    # ---------------------------------------------------------- demo tools
    def simulate_busy(self):
        self.holders.clear()
        for k in self.status:
            self.status[k] = random.choices(["Available", "Occupied", "Reserved", "Out of service"],
                                            [50, 30, 15, 5])[0]
        self.show_page("Dashboard")
        self.notice("Busy hours simulated", "Sample seat statuses have been updated.")

    def reset(self):
        self.status = {k: "Available" for k in all_seat_keys()}
        self.holders.clear()
        self.show_page("Dashboard")
        self.notice("Reset complete", "All demo seats are available again.")


if __name__ == "__main__":
    root = tk.Tk()
    app = SeatFinder(root)
    footer = tk.Frame(root, bg=BG)
    footer.place(relx=1.0, rely=1.0, anchor="se", x=-18, y=-8)
    tk.Button(footer, text="Reset demo seats", command=app.reset,
              bg=BG, fg=MUTED, activebackground=BG, activeforeground=GOLD,
              relief="flat", bd=0, font=("Segoe UI", 8), cursor="hand2").pack()
    root.mainloop()
