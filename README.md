# bot.py (полный бот с твоим кликером + авто-фокус + баланс-трекер + 2 минуты + СТАРАЯ ЛОГИКА)
import asyncio
import logging
import os
import re
import json
from datetime import datetime, timedelta
from typing import Dict, Any, Tuple, Optional
from dataclasses import dataclass, asdict
from pathlib import Path
from dotenv import load_dotenv
import webbrowser
import time

# ===== IMPORTS =====
try:
    import pyautogui
    import keyboard
    import pytesseract
    from PIL import ImageEnhance, ImageFilter
    AUTOCLICKER_AVAILABLE = True
    OCR_AVAILABLE = True
except ImportError as e:
    AUTOCLICKER_AVAILABLE = False
    OCR_AVAILABLE = False
    logging.warning("Библиотеки: pip install pyautogui keyboard pillow pytesseract python-dotenv PyGetWindow")

try:
    import pygetwindow as gw
    WINDOW_LIB_AVAILABLE = True
except ImportError:
    WINDOW_LIB_AVAILABLE = False

# ===== AI / CV IMPORTS =====
try:
    import cv2
    import numpy as np
    CV_AVAILABLE = True
except ImportError:
    CV_AVAILABLE = False
    logging.warning("OpenCV не установлен. AI-функции не будут работать. pip install opencv-python numpy")

# ===== SETTINGS =====
if AUTOCLICKER_AVAILABLE:
    pyautogui.FAILSAFE = True
    pyautogui.PAUSE = 0.2
else:
    pyautogui = None

# ===== АВТОМАТИЧЕСКАЯ НАСТРОЙКА ПОД ТВОЙ ЭКРАН (2880x1800) =====
if AUTOCLICKER_AVAILABLE:
    screen_width, screen_height = pyautogui.size()
    scale_x = screen_width / 1920
    scale_y = screen_height / 1080

    BASE_REGIONS = {
        "balance": (50, 50, 200, 30),
        "pair": (300, 180, 200, 40),
        "amount": (800, 600, 150, 40),
        "expiry": (800, 650, 100, 40),
        "buy_button": (700, 700, 120, 50),
        "sell_button": (900, 700, 120, 50),
    }

    REGIONS = {}
    for key, (x, y, w, h) in BASE_REGIONS.items():
        REGIONS[key] = (
            int(x * scale_x),
            int(y * scale_y),
            int(w * scale_x),
            int(h * scale_y)
        )
else:
    REGIONS = {
        "balance": (50, 50, 200, 30),
        "pair": (300, 180, 200, 40),
        "amount": (800, 600, 150, 40),
        "expiry": (800, 650, 100, 40),
        "buy_button": (700, 700, 120, 50),
        "sell_button": (900, 700, 120, 50),
    }

OCR_WHITELISTS = {
    "balance": "0123456789.$, ",
    "pair": "ABCDEFGHIJKLMNOPQRSTUVWXYZ/ ",
    "amount": "0123456789.,",
    "expiry": "0123456789MINUTESM ",
}

WINDOW_KEYWORDS = [
    "the most innovative trading platform",
    "pocket option",
    "pocketoption",
    "cabinet",
    "microsoft edge",
    "google chrome",
]

# ===== CONFIG & SECRETS =====
load_dotenv()
API_ID = int(os.getenv("API_ID", "34981096"))
API_HASH = os.getenv("API_HASH", "6a61c4ffe2c6ba1186b8d15c08abedea")
CHAT_ID = int(os.getenv("CHAT_ID", "5172749734"))

CONFIG_FILE = Path("config.json")

VALID_PAIRS = {
    "EURUSD", "GBPUSD", "USDJPY", "AUDUSD", "USDCAD", "USDCHF", "NZDUSD",
    "EURJPY", "GBPJPY", "EURGBP", "EURAUD", "EURCAD", "EURCHF",
    "GBPAUD", "GBPCAD", "GBPCHF", "AUDJPY", "AUDCAD", "AUDCHF",
    "CADJPY", "CHFJPY", "NZDJPY", "NZDCAD", "NZDCHF",
    "BHDUSD", "BHDJPY", "BHDGBP", "BHDCHF", "BHDNOK", "BHDCNY", "BHDSEK",
    "QARUSD", "QARJPY", "QARGBP", "QARCHF", "QARCNY",
    "KWDUSD", "KWDJPY", "KWDGBP", "KWDCHF", "KWDCNY",
    "AEDUSD", "AEDJPY", "AEDGBP", "AEDCHF", "AEDCNY", "AEDSEK", "AEDNOK",
    "SARUSD", "SARJPY", "SARGBP", "SARCHF", "SARCNY", "SARSEK", "SARNOK",
    "OMRUSD", "OMRJPY", "OMRGBP", "OMRCHF", "OMRCNY",
    "JODUSD", "JODJPY", "JODGBP", "JODCHF", "JODCNY",
    "MADUSD", "MADJPY", "MADGBP", "MADCHF", "MADCNY",
    "EGPUSD", "EGPJPY", "EGPGBP", "EGPCHF", "EGPCNY",
    "TRYUSD", "TRYJPY", "TRYGBP", "TRYCHF", "TRYCNY",
    "ZARUSD", "ZARJPY", "ZARGBP", "ZARCHF", "ZARCNY",
    "SGDUSD", "SGDJPY", "SGDGBP", "SGDCHF", "SGDCNY",
    "HKDUSD", "HKDJPY", "HKDGBP", "HKDCHF", "HKDCNY",
    "THBUSD", "THBJPY", "THBGBP", "THBCHF", "THBCNY",
    "MYRUSD", "MYRJPY", "MYRGBP", "MYRCHF", "MYRCNY",
    "IDRUSD", "IDRJPY", "IDRGBP", "IDRCHF", "IDRCNY",
    "PHPUSD", "PHPJPY", "PHPGBP", "PHPCHF", "PHPCNY",
    "VNDUSD", "VNDJPY", "VNDGBP", "VNDCHF", "VNDCNY",
    "INRUSD", "INRJPY", "INRGBP", "INRCHF", "INRCNY",
    "PKRUSD", "PKRJPY", "PKRGBP", "PKRCHF", "PKRCNY",
    "BDTUSD", "BDTJPY", "BDTGBP", "BDTCHF", "BDTCNY",
    "LKRUSD", "LKRJPY", "LKRGBP", "LKRCHF", "LKRCNY",
    "USDSGD", "USDHKD", "USDTWD", "USDMXN", "USDBRL", "USDCZK", "USDHUF", "USDPLN", "USDRUB",
}

PAIR_RE = re.compile(r"pair:\s*([A-Z]{3,}/[A-Z]{3,}|[A-Z]{3,})", re.IGNORECASE)
DURATION_RE = re.compile(r"(\d+)\s*(minute|minutes|min|M|MIN)", re.IGNORECASE)

# ===== CONFIG DATA CLASS (ВОССТАНОВЛЕН ОРИГИНАЛ) =====
@dataclass
class Config:
    base_risk_percent: float = 1.0
    martingale_multiplier: float = 2.0
    max_streak: int = 5
    default_balance: float = 100.0
    dry_run: bool = False
    focus_delay: float = 0.2
    field_delay: float = 0.4
    validate_delay: float = 0.8
    pair_load_delay: float = 2.0
    login_delay: float = 3.0
    mode: str = "fast"
    window_title: str = "The Most Innovative Trading Platform"
    regions: Optional[Dict[str, Tuple[int, int, int, int]]] = None
    pair_format: str = "EURUSD"
    hotkeys: Dict[str, Dict[str, str]] = None

def load_config() -> Config:
    if CONFIG_FILE.exists():
        with open(CONFIG_FILE, 'r') as f:
            data = json.load(f)
            filtered_data = {k: v for k, v in data.items() if k in Config.__dataclass_fields__}
            config = Config(**filtered_data)
    else:
        config = Config()
        save_config(config)
    
    if config.regions is None:
        config.regions = REGIONS
    if config.hotkeys is None:
        config.hotkeys = {
            "EURUSD": {"BUY": "f1", "SELL": "f2"},
            "GBPUSD": {"BUY": "f3", "SELL": "f4"},
        }
    save_config(config)
    return config

def save_config(config: Config):
    with open(CONFIG_FILE, 'w') as f:
        json.dump(asdict(config), f, indent=2)

config = load_config()

# ===== LOGGING =====
logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s",
                    handlers=[logging.FileHandler("bot.log", encoding="utf-8"), logging.StreamHandler()])
logger = logging.getLogger(__name__)

# ===== TELEGRAM =====
from telethon import TelegramClient, events
client = TelegramClient("session", API_ID, API_HASH)

# ===== OCR SETUP =====
if OCR_AVAILABLE:
    tesseract_path = os.getenv("TESSERACT_CMD")
    if tesseract_path:
        pytesseract.pytesseract.tesseract_cmd = tesseract_path
    else:
        logger.warning("Переменная окружения TESSERACT_CMD не установлена.")

# ===== STATE =====
seen_signals: Dict[str, datetime] = {}
mg_state: Dict[str, Any] = {
    "multiplier": 1.0, 
    "streak_losses": 0, 
    "last_pair": None,
    "pending_pair": None,
    "pending_duration": None,
    "pending_otc": False,
    "balance": config.default_balance,
    "base_risk_percent": config.base_risk_percent,
}
state_lock = asyncio.Lock()

# ===== АВТО-ФОКУС НА БРАУЗЕР =====
def focus_browser():
    """Автоматически фокусируется на окне браузера Pocket Option"""
    try:
        # Ищем окна с "Pocket" в названии
        windows = gw.getWindowsWithTitle("Pocket")
        if not windows:
            # Если не нашли "Pocket", ищем "Edge" или "Chrome"
            windows = gw.getWindowsWithTitle("Edge")
            if not windows:
                windows = gw.getWindowsWithTitle("Chrome")
        
        if windows:
            win = windows[0]
            win.activate()
            time.sleep(1)  # Ждём 1 секунду для активации
            logger.info("✅ Браузер сфокусирован: %s", win.title)
            return True
        else:
            logger.warning("⚠️ Браузер не найден")
            return False
    except Exception as e:
        logger.error(f"❌ Ошибка авто-фокуса: {e}")
        return False

# ===== ТВОЙ РАБОЧИЙ КЛИКЕР (БЕЗ ИЗМЕНЕНИЙ) =====
import cv2
import numpy as np
import pyautogui
import time
import logging
from pathlib import Path
import pytesseract
import re
from datetime import datetime

try:
    import keyboard
except ImportError:
    keyboard = None

try:
    import pygetwindow as gw
except ImportError:
    gw = None

# Путь к Tesseract
pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger(__name__)

pyautogui.FAILSAFE = True
pyautogui.PAUSE = 0.2


class OrcAgent:
    def __init__(self, assets_dir="assets", debug_dir="debug"):
        self.assets_dir = Path(assets_dir)
        self.debug_dir = Path(debug_dir)
        self.debug_dir.mkdir(exist_ok=True)
        self.stop_requested = False

        # Шаблоны для кликов
        self.templates = {
            "pair_input.png": {"threshold": 0.60, "grayscale": True},
            "stake_input.png": {"threshold": 0.60, "grayscale": True},
            "buy_button.png": {"threshold": 0.68, "grayscale": False},
            "sell_button.png": {"threshold": 0.68, "grayscale": False},
        }

    def install_hotkeys(self):
        if keyboard is not None:
            keyboard.add_hotkey("F9", self.request_stop)
            logger.info("F9 hotkey registered")

    def request_stop(self):
        self.stop_requested = True
        logger.warning("F9 pressed, stopping")

    def check_stop(self):
        if self.stop_requested:
            raise KeyboardInterrupt("Stopped by F9")

    def validate_assets(self):
        if not self.assets_dir.exists():
            raise FileNotFoundError(f"Folder not found: {self.assets_dir.resolve()}")

        missing = []
        for name in self.templates:
            if not (self.assets_dir / name).exists():
                missing.append(name)

        if missing:
            raise FileNotFoundError("Missing templates: " + ", ".join(missing))

    def capture_screen(self):
        self.check_stop()
        img = pyautogui.screenshot()
        rgb = np.array(img)
        bgr = cv2.cvtColor(rgb, cv2.COLOR_RGB2BGR)
        return bgr

    def save_image(self, name, image):
        path = self.debug_dir / name
        cv2.imwrite(str(path), image)
        return path

    def load_template(self, template_name):
        path = self.assets_dir / template_name
        template = cv2.imread(str(path))
        if template is None:
            raise RuntimeError(f"Failed to read template: {path}")
        return template

    def run_match(self, screen_bgr, template_bgr, grayscale=False):
        if grayscale:
            screen_src = cv2.cvtColor(screen_bgr, cv2.COLOR_BGR2GRAY)
            template_src = cv2.cvtColor(template_bgr, cv2.COLOR_BGR2GRAY)
        else:
            screen_src = screen_bgr
            template_src = template_bgr

        result = cv2.matchTemplate(screen_src, template_src, cv2.TM_CCOEFF_NORMED)
        return result

    def top_candidates(self, result, template_size, k=5, min_distance=60):
        flat = result.flatten()
        idxs = np.argpartition(flat, -k * 10)[-k * 10:]
        idxs = idxs[np.argsort(flat[idxs])[::-1]]

        h, w = template_size
        candidates = []

        for idx in idxs:
            score = float(flat[idx])
            y, x = np.unravel_index(idx, result.shape)
            cx = int(x + w // 2)
            cy = int(y + h // 2)

            too_close = False
            for c in candidates:
                px, py = c["center"]
                if abs(cx - px) < min_distance and abs(cy - py) < min_distance:
                    too_close = True
                    break

            if too_close:
                continue

            candidates.append({
                "score": score,
                "top_left": (int(x), int(y)),
                "center": (cx, cy),
                "size": (int(w), int(h)),
            })

            if len(candidates) >= k:
                break

        return candidates

    def locate_best(self, screen_bgr, template_name):
        self.check_stop()

        cfg = self.templates[template_name]
        template = self.load_template(template_name)

        color_result = self.run_match(screen_bgr, template, grayscale=False)
        gray_result = self.run_match(screen_bgr, template, grayscale=True)

        color_candidates = self.top_candidates(color_result, template.shape[:2], k=5)
        gray_candidates = self.top_candidates(gray_result, template.shape[:2], k=5)

        best_color = color_candidates[0] if color_candidates else None
        best_gray = gray_candidates[0] if gray_candidates else None

        use_gray = False
        if best_color and best_gray:
            use_gray = best_gray["score"] > best_color["score"]
        elif best_gray and not best_color:
            use_gray = True

        selected = best_gray if use_gray else best_color
        mode = "grayscale" if use_gray else "color"
        all_candidates = gray_candidates if use_gray else color_candidates

        if selected is None:
            return {
                "template": template_name,
                "mode": mode,
                "threshold": cfg["threshold"],
                "ok": False,
                "best": None,
                "candidates": [],
                "heatmap": gray_result if use_gray else color_result,
            }

        return {
            "template": template_name,
            "mode": mode,
            "threshold": cfg["threshold"],
            "ok": selected["score"] >= cfg["threshold"],
            "best": selected,
            "candidates": all_candidates,
            "heatmap": gray_result if use_gray else color_result,
        }

    # ===== НОВЫЙ click_element С НОВЫМ СКРИНОМ =====
    def click_element(self, template_name, region=None):
        """Кликает по элементу по свежему скриншоту"""
        screen_bgr = self.capture_screen()

        offset_x = 0
        offset_y = 0

        if region is not None:
            x, y, w, h = region
            screen_bgr = screen_bgr[y:y+h, x:x+w]
            offset_x = x
            offset_y = y

        det = self.locate_best(screen_bgr, template_name)
        if det["ok"] and det["best"]:
            cx, cy = det["best"]["center"]
            cx += offset_x
            cy += offset_y
            pyautogui.click(cx, cy)
            logger.info(f"🖱️ Клик по {template_name} в ({cx}, {cy})")
            return True

        logger.warning(f"❌ Не удалось кликнуть по {template_name}")
        return False

    # ===== ОЧИСТКА ПОЛЯ И ВВОД =====
    def clear_and_type(self, text: str):
        pyautogui.hotkey("ctrl", "a")
        time.sleep(0.1)
        pyautogui.press("backspace")
        time.sleep(0.1)
        pyautogui.typewrite(str(text))
        logger.info(f"⌨️ Введено: {text}")

    # ===== НОВАЯ ФУНКЦИЯ ПОИСКА PNG =====
    def find_template(self, screen, template_path, threshold=0.6):
        """Ищет шаблон на экране. Возвращает центр, если найден."""
        # Попытка 1: OpenCV
        template = cv2.imread(template_path)
        if template is None:
            logger.error(f"[ERROR] Не найден файл: {template_path}")
            return None

        result = cv2.matchTemplate(screen, template, cv2.TM_CCOEFF_NORMED)
        _, max_val, _, max_loc = cv2.minMaxLoc(result)

        if max_val >= threshold:
            h, w = template.shape[:2]
            logger.info(f"[OK] SEARCH найден через OpenCV {max_loc} conf={max_val:.2f}")
            return {
                "type": "search_box",
                "center": (max_loc[0] + w // 2, max_loc[1] + h // 2),
                "box": (max_loc[0], max_loc[1], w, h),
                "confidence": max_val
            }

        # Попытка 2: pyautogui.locateOnScreen (если OpenCV не нашёл)
        logger.warning("⚠️ OpenCV не нашёл search_box — пробуем pyautogui...")
        try:
            box = pyautogui.locateOnScreen(template_path, confidence=0.6, grayscale=True)
            if box:
                center = pyautogui.center(box)
                logger.info(f"[OK] SEARCH найден через pyautogui {center}")
                return {
                    "type": "search_box",
                    "center": center,
                    "box": box,
                    "confidence": 0.6
                }

            # Попытка 3: с region
            logger.warning("⚠️ pyautogui не нашёл search_box — пробуем с region...")
            box = pyautogui.locateOnScreen(
                template_path,
                region=(200, 200, 1200, 500),
                confidence=0.45,
                grayscale=True
            )
            if box:
                center = pyautogui.center(box)
                logger.info(f"[OK] SEARCH найден через pyautogui с region {center}")
                return {
                    "type": "search_box",
                    "center": center,
                    "box": box,
                    "confidence": 0.45
                }
        except Exception as e:
            logger.error(f"Ошибка pyautogui.locateOnScreen: {e}")

        logger.error("❌ SEARCH не найден ни через OpenCV, ни через pyautogui")
        return None

    # ===== ОСНОВНАЯ ФУНКЦИЯ ТОРГОВЛИ =====
    def trade(self, pair: str, direction: str, stake: float, duration: int, otc: bool = False):
        """
        Выполняет полную торговлю:
        1. Клик на заголовок -> Поиск search_box.png -> Ввод пары
        2. Если OTC — ищем и кликаем otc_label.png
        3. Ввод суммы (без клика по времени)
        4. Клик по кнопке BUY/SELL
        """
        # ДОБАВЛЯЕМ АВТО-ФОКУС
        focus_browser()
        
        logger.info(f"🚀 TRADE: {direction} {pair} {'OTC' if otc else ''} {duration}M stake=${stake}")

        # 1. Ввод пары
        logger.info("🔍 Ввод пары...")

        # Шаг 1: Клик на заголовок (чтобы открыть меню)
        clicked = self.click_element("pair_input.png")
        if not clicked:
            logger.warning("⚠️ Шаблон pair_input.png не найден. Пробуем кликнуть по координатам (250, 240).")
            pyautogui.click(250, 240)

        time.sleep(0.6)  # Ждем, пока откроется меню

        # Шаг 2: Ищем search_box.png
        logger.info("🔍 Ищем search_box.png...")
        # Делаем скриншот для поиска search_box
        screen = self.capture_screen()
        search_box = self.find_template(screen, "assets/search_box.png", threshold=0.6)

        if search_box:
            logger.info(f"[OK] SEARCH найден {search_box['center']} conf={search_box['confidence']:.2f}")
            x, y = search_box["center"]
            pyautogui.click(x, y)
        else:
            logger.warning("[X] SEARCH не найден. Пробуем кликнуть по координатам (450, 300).")
            pyautogui.click(450, 300)  # Запасные координаты

        time.sleep(0.4)  # Ждем, пока поле активируется

        # Вводим только пару (без OTC)
        pyautogui.typewrite(pair.replace("/", ""))
        pyautogui.press('enter')
        time.sleep(0.5)

        # 2. Если OTC — ищем и кликаем otc_label.png
        if otc:
            logger.info("🔍 Ищем otc_label.png...")
            otc_found = False
            try:
                # Попытка 1: точный поиск
                otc_box = pyautogui.locateOnScreen(
                    "assets/otc_label.png",
                    confidence=0.7,
                    grayscale=False
                )
                if otc_box:
                    x, y = pyautogui.center(otc_box)
                    logger.info(f"✅ OTC найден (точно) в ({x}, {y})")
                    # Кликаем точно в центр
                    pyautogui.click(x, y)  # 👈 КЛИКАЕМ В ЦЕНТР
                    otc_found = True
                else:
                    logger.warning("⚠️ OTC не найден (точно)")

                # Попытка 2: с region
                if not otc_found:
                    otc_box = pyautogui.locateOnScreen(
                        "assets/otc_label.png",
                        region=(300, 200, 800, 400),
                        confidence=0.5,
                        grayscale=False
                    )
                    if otc_box:
                        x, y = pyautogui.center(otc_box)
                        logger.info(f"✅ OTC найден (регион) в ({x}, {y})")
                        pyautogui.click(x, y)  # 👈 КЛИКАЕМ В ЦЕНТР
                        otc_found = True
                    else:
                        logger.warning("⚠️ OTC не найден (регион)")

            except Exception as e:
                logger.error(f"Ошибка поиска OTC: {e}")

            # Если нашли — нажимаем Enter дважды
            if otc_found:
                pyautogui.press("enter")
                time.sleep(0.3)
                pyautogui.press("enter")
                time.sleep(0.5)
            else:
                logger.error("❌ OTC не найден — пропускаем")
                time.sleep(0.5)
        else:
            time.sleep(0.5)

        # 3. Ввод суммы (без клика по времени)
        logger.info("🔍 Ввод суммы...")
        # Ищем сумму только в правой панели
        stake_region = (2350, 250, 400, 500)   # подгони под свой экран
        if self.click_element("stake_input.png", region=stake_region):
            time.sleep(0.6)
            self.clear_and_type(stake)
            time.sleep(0.5)
        else:
            logger.error("❌ Не удалось кликнуть по полю ставки")

        # 4. Клик по кнопке
        btn_template = "buy_button.png" if direction.upper() == "BUY" else "sell_button.png"
        logger.info(f"🔍 Клик по кнопке {direction}...")
        button_region = (2350, 700, 400, 500)   # правая нижняя часть
        if self.click_element(btn_template, region=button_region):
            logger.info("✅ Торговля выполнена")
        else:
            logger.error("❌ Не удалось кликнуть по кнопке")

    # ===== БЫСТРЫЙ БАЛАНС (ТВОЙ КОД) =====
    def get_balance_fast(self):
        """Быстро получает баланс через OCR"""
        try:
            screenshot = pyautogui.screenshot(region=(2300, 230, 300, 120))
            img = cv2.cvtColor(np.array(screenshot), cv2.COLOR_RGB2BGR)

            gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
            gray = cv2.resize(gray, None, fx=4, fy=4)
            _, thresh = cv2.threshold(gray, 140, 255, cv2.THRESH_BINARY)

            text = pytesseract.image_to_string(
                thresh,
                config="--psm 7 -c tessedit_char_whitelist=0123456789."
            )

            print("OCR:", text)

            match = re.search(r"\d+\.\d+", text)
            if match:
                return float(match.group())

            return None
        except Exception as e:
            logger.error(f"Ошибка при получении баланса: {e}")
            return None

    # ===== НОВАЯ ФУНКЦИЯ: ЖДЁМ БРАУЗЕР =====
    def wait_for_browser_focus(self, timeout=15):
        """Ждёт, пока пользователь откроет браузер с PocketOption"""
        import time
        import pygetwindow as gw

        start = time.time()
        while time.time() - start < timeout:
            win = gw.getActiveWindow()
            title = win.title if win else ""
            print("ACTIVE WINDOW:", repr(title))

            if "Edge" in title or "PocketOption" in title or "Trading" in title:
                logger.info("✅ Браузер найден: %s", title)
                return True

            time.sleep(1)

        logger.error("❌ Браузер не найден за %d секунд", timeout)
        return False

# ===== BALANCE TRACKER (каждые 10 секунд) =====
async def balance_tracker():
    """Показывает баланс каждые 10 секунд"""
    bot = OrcAgent(assets_dir="assets", debug_dir="debug")
    while True:
        try:
            balance = bot.get_balance_fast()
            if balance is not None:
                logger.info(f"💰 БАЛАНС: ${balance:.2f}")
            else:
                logger.warning("❌ Не удалось получить баланс")
        except Exception as e:
            logger.error(f"Ошибка баланс-трекера: {e}")
        await asyncio.sleep(10)  # Каждые 10 секунд

# ===== UTILS =====
async def async_sleep(delay: float):
    await asyncio.sleep(delay)

def normalize_pair_value(pair_str: str) -> str:
    return pair_str.replace("/", "").replace(" ", "").upper()

def format_pair_for_output(pair: str, with_slash: bool = False) -> str:
    pair = pair.replace("/", "").replace(" ", "").upper()
    if len(pair) != 6:
        return pair
    return f"{pair[:3]}/{pair[3:]}" if with_slash else pair

def extract_pair_from_text(text: str) -> Dict[str, Any]:
    if not text:
        return {"pair": None, "otc": False}
    upper = text.upper()
    otc = "OTC" in upper
    pair_match = PAIR_RE.search(text)
    if pair_match:
        pair_str = pair_match.group(1)
        pair = normalize_pair_value(pair_str)
        if pair in VALID_PAIRS:
            return {"pair": pair, "otc": otc}
        if len(pair) == 6:
            return {"pair": pair, "otc": otc}
    return {"pair": None, "otc": otc}

def extract_duration(text: str) -> Optional[int]:
    duration_match = DURATION_RE.search(text)
    if duration_match:
        return int(duration_match.group(1))
    return None

def get_balance() -> float:
    bot = OrcAgent(assets_dir="assets", debug_dir="debug")
    balance = bot.get_balance_fast()
    if balance is not None:
        return balance
    logger.warning("OCR баланс failed, fallback=%.2f", config.default_balance)
    return config.default_balance

def calc_stake(balance: float, mult: float = 1.0) -> float:
    return round(balance * (mg_state["base_risk_percent"] / 100) * mult, 2)

async def validate_field(region_key: str, expected: str, max_tries: int = 3) -> bool:
    logger.info(f"⏭️  Пропуск OCR проверки для {region_key} (режим FAST)")
    return True

def extract_first_number(text: str) -> float:
    if not text:
        return 0.0
    nums = re.findall(r'\d+\.?\d*', text.replace(',', '.'))
    if not nums:
        return 0.0
    try:
        return float(nums[0])
    except ValueError:
        return 0.0

# ===== PARSER (СТАРАЯ ЛОГИКА) =====
def parse_message(text: str) -> Dict[str, Any]:
    logger.info(f"🔍 АНАЛИЗ СООБЩЕНИЯ: {text}")
    upper = text.upper()
    
    pair_info = extract_pair_from_text(text)
    duration = extract_duration(text)
    
    logger.info(f"📊 Пара: {pair_info}, Время: {duration}")
    
    if "WE ARE WAITING FOR A SIGNAL" in upper:
        if pair_info["pair"] and duration:
            return {"type": "preparing_1x", "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}

    if "SIGNAL" in upper and ("BUY🟢" in upper or "SELL🔴" in upper) and "(MARTINGALE" not in upper:
        if pair_info["pair"] and duration:
            is_buy = "BUY" in upper
            is_sell = "SELL" in upper
            if is_buy:
                return {"type": "signal", "direction": "BUY", "martingale": False, "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}
            if is_sell:
                return {"type": "signal", "direction": "SELL", "martingale": False, "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}

    if "WAITING FOR A SIGNAL IN 2 MINUTE" in upper and "ANALYZING ENTRY POINT" in upper:
        if pair_info["pair"] and duration:
            return {"type": "preparing_2x", "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}

    if "SIGNAL" in upper and "(MARTINGALE" in upper:
        if pair_info["pair"] and duration:
            is_buy = "BUY" in upper
            is_sell = "SELL" in upper
            if is_buy:
                return {"type": "signal", "direction": "BUY", "martingale": True, "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}
            if is_sell:
                return {"type": "signal", "direction": "SELL", "martingale": True, "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}

    if "SIGNAL RESULT:" in upper:
        if "WIN" in upper:
            return {"type": "result", "result": "WIN", "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}
        if "LOSS" in upper:
            return {"type": "result", "result": "LOSS", "pair": pair_info["pair"], "duration": duration, "otc": pair_info["otc"]}

    logger.info(f"📝 Не распознано (мусор): {text}")
    return {"type": "ignore"}

def build_key(parsed: Dict[str, Any]) -> str:
    return "|".join([
        str(parsed.get("type", "")),
        str(parsed.get("pair", "")),
        str(parsed.get("duration", "")),
        str(parsed.get("direction", "")),
        str(parsed.get("martingale", False)),
        str(parsed.get("result", "")),
    ])

def is_duplicate(key: str, ttl: int = 120) -> bool:
    cleanup_seen()
    now = datetime.now()
    if key in seen_signals and now < seen_signals[key]:
        return True
    seen_signals[key] = now + timedelta(seconds=ttl)
    return False

def cleanup_seen():
    now = datetime.now()
    expired = [k for k, v in seen_signals.items() if now >= v]
    for k in expired:
        del seen_signals[k]

# ===== HANDLERS =====
@client.on(events.NewMessage)
async def new_msg(event):
    if event.chat_id != CHAT_ID:
        return
        
    try:
        text = event.raw_text or ""
        if not text.strip():
            return

        logger.info("="*60)
        logger.info("📥 НОВОЕ СООБЩЕНИЕ")
        logger.info("💬 Текст: %s", text)
        logger.info("="*60)
        
        parsed = parse_message(text)
        logger.info("🔍 Результат анализа: %s", parsed)
        
        if parsed["type"] == "ignore":
            logger.warning("⚠️ Сообщение игнорировано (мусор)")
            return

        key = build_key(parsed)
        if is_duplicate(key):
            logger.info("🔁 Дубликат сигнала, пропущен")
            return

        logger.info("📱 ОБРАБОТКА СИГНАЛА")

        if parsed["type"] in ["preparing_1x", "preparing_2x"]:
            await handle_preparing(parsed)
        elif parsed["type"] == "signal":
            await handle_signal(parsed)
        elif parsed["type"] == "result":
            await handle_result(parsed)

    except Exception as e:
        logger.exception("❌ Ошибка обработчика: %s", e)

@client.on(events.MessageEdited)
async def edited_msg(event):
    if event.chat_id != CHAT_ID:
        return
    try:
        text = event.raw_text or ""
        if not text.strip():
            return

        logger.info("="*60)
        logger.info("📥 ОТРЕДАКТИРОВАННОЕ СООБЩЕНИЕ")
        logger.info("💬 Текст: %s", text)
        logger.info("="*60)
        
        parsed = parse_message(text)
        logger.info("🔍 Результат анализа: %s", parsed)
        
        if parsed["type"] == "ignore":
            logger.warning("⚠️ Сообщение игнорировано")
            return

        key = build_key(parsed)
        if is_duplicate(key):
            logger.info("🔁 Дубликат сигнала, пропущен")
            return

        logger.info("📱 ОБРАБОТКА СИГНАЛА")

        if parsed["type"] in ["preparing_1x", "preparing_2x"]:
            await handle_preparing(parsed)
        elif parsed["type"] == "signal":
            await handle_signal(parsed)
        elif parsed["type"] == "result":
            await handle_result(parsed)

    except Exception as e:
        logger.exception("❌ Ошибка обработчика: %s", e)

async def handle_preparing(parsed: Dict[str, Any]):
    async with state_lock:
        logger.info(f"⚙️ Обработка подготовки типа: {parsed.get('type')}")
        
        balance = get_balance()
        mg_state["balance"] = balance

        if parsed.get("type") == "preparing_1x":
            logger.info("⏳ ПОДГОТОВКА 1X: %s %dM", parsed["pair"], parsed["duration"])
            mg_state["pending_pair"] = parsed["pair"]
            mg_state["pending_duration"] = parsed["duration"]
            mg_state["pending_otc"] = parsed["otc"]
            stake = calc_stake(balance, 1.0)
            mg_state["prepared_stake"] = stake
            mg_state["prepared_multiplier"] = 1.0
            logger.info("✅ Подготовлено 1x: stake=$%.2f", stake)
            bot = OrcAgent(assets_dir="assets", debug_dir="debug")
            bot.trade(parsed["pair"], "BUY", 0.01, parsed["duration"], otc=parsed["otc"])
            return

        if parsed.get("type") == "preparing_2x":
            logger.info("⏳ ПОДГОТОВКА 2X: %s %dM", parsed["pair"], parsed["duration"])
            mg_state["pending_pair"] = parsed["pair"]
            mg_state["pending_duration"] = parsed["duration"]
            mg_state["pending_otc"] = parsed["otc"]
            stake = calc_stake(balance, 2.0)
            mg_state["prepared_stake"] = stake
            mg_state["prepared_multiplier"] = 2.0
            logger.info("✅ Подготовлено 2x: stake=$%.2f", stake)
            bot = OrcAgent(assets_dir="assets", debug_dir="debug")
            bot.trade(parsed["pair"], "BUY", 0.01, parsed["duration"], otc=parsed["otc"])
            return

async def handle_signal(parsed: Dict[str, Any]):
    async with state_lock:
        logger.info(f"⚙️ Обработка сигнала типа: {parsed.get('type')}")
        
        balance = get_balance()
        mg_state["balance"] = balance

        if mg_state["streak_losses"] >= config.max_streak:
            logger.error("🛑 Макс серия лоссов! Стоп.")
            return
        
        stake = mg_state.get("prepared_stake", calc_stake(balance))
        
        otc_str = " (OTC)" if parsed.get("otc") else ""
        logger.info(f"🔵 СИГНАЛ {parsed['direction']} {parsed['pair']}{otc_str} {parsed['duration']}M | баланс=${balance:.2f} | ставка=${stake:.2f} | mult={mg_state['multiplier']:.2f}x")
        
        bot = OrcAgent(assets_dir="assets", debug_dir="debug")
        success = bot.trade(parsed["pair"], parsed["direction"], stake, parsed["duration"], otc=parsed["otc"])
        
        if success:
            logger.info("✅ Торговля выполнена")
        else:
            logger.error("❌ Торговля не удалась")

        mg_state["prepared_stake"] = None
        mg_state["prepared_multiplier"] = 1.0

async def handle_result(parsed: Dict[str, Any]):
    async with state_lock:
        otc_str = " (OTC)" if parsed.get("otc") else ""
        if parsed["result"] == "WIN":
            mg_state["balance"] = round(mg_state["balance"] * 1.01, 2)
            mg_state["multiplier"] = 1.0
            mg_state["streak_losses"] = 0
            logger.info("✅ WIN | %s %sM%s | 🔄 RESET 1%% | balance=$%.2f", parsed["pair"], parsed["duration"], otc_str, mg_state["balance"])
        elif parsed["result"] == "LOSS":
            mg_state["balance"] = round(mg_state["balance"] * 1.01, 2)
            mg_state["multiplier"] = 1.0
            mg_state["streak_losses"] = 0
            logger.info("❌ LOSS | %s %sM%s | 🔄 RESET 1%% | balance=$%.2f", parsed["pair"], parsed["duration"], otc_str, mg_state["balance"])
        else:
            logger.info(
                "❓ %s | %s %sM%s | mult=%.2fx | streak=%d | balance=$%.2f",
                parsed["result"],
                parsed["pair"],
                parsed["duration"],
                otc_str,
                mg_state["multiplier"],
                mg_state["streak_losses"],
                mg_state["balance"]
            )

# ===== АВТОМАТИЧЕСКОЕ ОТКРЫТИЕ САЙТА =====
def open_site_automatically():
    """Автоматически открывает сайт и проверяет загрузку"""
    logger.info("🌐 Открываем сайт: https://pocketoption.com/en/cabinet/")
    try:
        webbrowser.open("https://pocketoption.com/en/cabinet/")
        logger.info("⏳ Ждём загрузки сайта (15 секунд)...")
        time.sleep(15)
        
        if WINDOW_LIB_AVAILABLE:
            window = find_target_window()
            if window:
                logger.info("✅ Сайт открыт и активен!")
                return True
            else:
                logger.warning("⚠️ Сайт открыт, но окно не найдено. Проверьте, что браузер запущен.")
                return True
        else:
            logger.warning("⚠️ PyGetWindow недоступен, но сайт должен открыться.")
            return True
    except Exception as e:
        logger.error(f"❌ Ошибка при открытии сайта: {e}")
        return False

def find_target_window():
    titles = gw.getAllTitles()
    for title in titles:
        low = (title or "").lower()
        if any(k in low for k in WINDOW_KEYWORDS):
            windows = gw.getWindowsWithTitle(title)
            if windows:
                return windows[0]
    return None

# ===== MAIN =====
async def main():
    await client.start()
    logger.info("="*60)
    logger.info("🚀 БОТ ЗАПУЩЕН (AI-AGENT V8.2 - STABLE + ORC AGENT + AUTO OPEN + BALANCE TRACKER)")
    logger.info("⚙️  Версия: v8.2 (ZENBOOK 2880x1800 FIX + NO REGIONS BREAK)")
    logger.info("🆔 Chat ID: %d", CHAT_ID)
    logger.info("🛡️  Max streak: %d", config.max_streak)
    logger.info("💰 OCR: %s | Автокликер: %s | AI: %s", OCR_AVAILABLE, AUTOCLICKER_AVAILABLE, CV_AVAILABLE)
    logger.info("📂 Config file: %s", CONFIG_FILE)
    logger.info("🖥️ Window Title: %s", config.window_title)
    logger.info("🧪 Dry Run: %s", config.dry_run)
    
    if AUTOCLICKER_AVAILABLE:
        screen_w, screen_h = pyautogui.size()
        logger.info("🖥️ Экран: %dx%d", screen_w, screen_h)
        logger.info("📍 Адаптированные координаты: %s", REGIONS)
    logger.info("="*60)
    
    # АВТОМАТИЧЕСКОЕ ОТКРЫТИЕ САЙТА
    if not open_site_automatically():
        logger.error("❌ Не удалось открыть сайт. Бот остановлен.")
        return
    
    logger.info("✅ Сайт открыт. Бот готов к работе!")
    
    # ЗАПУСК БАЛАНС-ТРЕКЕРА
    asyncio.create_task(balance_tracker())
    logger.info("📊 Баланс-трекер запущен (каждые 10 секунд)")
    
    await client.run_until_disconnected()

if __name__ == "__main__":
    asyncio.run(main())
