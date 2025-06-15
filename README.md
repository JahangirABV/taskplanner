import speech_recognition as sr
import re
from datetime import datetime, timedelta
import json
import os

recognizer = sr.Recognizer()

def get_voice_input():
    try:
        with sr.Microphone() as source:
            print("Скажите задачу (например, 'Урок по математике, срок 20 июня, примерно 2 часа') или 'удалить [задача]'...")
            recognizer.adjust_for_ambient_noise(source, duration=3)
            audio = recognizer.listen(source, timeout=15, phrase_time_limit=30)
            text = recognizer.recognize_google(audio, language="ru-RU")
            print(f"Распознано: {text}")
            return text
    except (sr.WaitTimeoutError, sr.UnknownValueError, sr.RequestError):
        print("Ошибка голосового ввода. Введите задачу вручную.")
    except KeyboardInterrupt:
        print("\nПрограмма завершена.")
        return "готово"

    return input("Введите задачу (или 'готово' для завершения): ").strip()


def parse_task(text):
    if not text or text.lower() == "готово":
        return None

    print(f"Парсинг текста: {text}")

    if text.lower().startswith("удалить"):
        return {"action": "delete", "task": text[len("удалить"):].strip()}

    task_match = re.match(r"^(.*?)(?:,\s*срок\s*|\s*до\s*)?", text, re.IGNORECASE)
    deadline_match = re.search(
        r"(?:срок|до)\s*(\d{1,2}\s*(января|февраля|марта|апреля|мая|июня|июля|августа|сентября|октября|ноября|декабря))",
        text, re.IGNORECASE)
    time_match = re.search(r"примерно\s*([\d.,]+)\s*(?:часа|часов|час)", text, re.IGNORECASE)
    relative_match = re.search(r"(завтра|через\s+(\d+)\s+д(ня|ней|ень))", text, re.IGNORECASE)

    task = task_match.group(1).strip() if task_match and task_match.group(1).strip() else text.split(",")[0].strip()
    deadline = None
    hours = float(time_match.group(1).replace(',', '.')) if time_match else 1.0

    if deadline_match:
        try:
            deadline = datetime.strptime(f"{deadline_match.group(1)} 2025", "%d %B %Y")
        except ValueError:
            deadline = None
    elif relative_match:
        if "завтра" in relative_match.group(1).lower():
            deadline = datetime.now() + timedelta(days=1)
        elif relative_match.group(2):
            days = int(relative_match.group(2))
            deadline = datetime.now() + timedelta(days=days)

    return {
        "task": task,
        "deadline": deadline.strftime("%Y-%m-%d") if deadline else None,
        "estimated_hours": round(hours, 2)
    }


def save_tasks(tasks):
    with open("tasks.json", "w", encoding="utf-8") as f:
        json.dump([t for t in tasks if "action" not in t], f, ensure_ascii=False, indent=4)


def load_tasks():
    if os.path.exists("tasks.json"):
        with open("tasks.json", "r", encoding="utf-8") as f:
            return json.load(f)
    return []


def generate_daily_plan(tasks, today):
    plan = []
    today_str = today.strftime('%d %B').replace('January', 'января').replace('February', 'февраля')\
        .replace('March', 'марта').replace('April', 'апреля').replace('May', 'мая')\
        .replace('June', 'июня').replace('July', 'июля').replace('August', 'августа')\
        .replace('September', 'сентября').replace('October', 'октября')\
        .replace('November', 'ноября').replace('December', 'декабря')

    for task in tasks:
        deadline_str = task.get("deadline")
        estimated = task.get("estimated_hours", 1.0)
        task_name = task.get("task", "Без названия")

        if deadline_str:
            try:
                deadline = datetime.strptime(deadline_str, "%Y-%m-%d")
                days_left = (deadline - today).days
                if days_left < 0:
                    plan.append(f"{today_str}: {task_name}, {estimated} {get_hour_word(estimated)} (просрочено)")
                elif days_left == 0:
                    plan.append(f"{today_str}: {task_name}, {estimated} {get_hour_word(estimated)} (дедлайн сегодня)")
                else:
                    hours_today = estimated / days_left
                    plan.append(f"{today_str}: {task_name}, {hours_today:.1f} {get_hour_word(hours_today)}")
            except Exception:
                plan.append(f"{today_str}: {task_name}, {estimated} {get_hour_word(estimated)}")
        else:
            plan.append(f"{today_str}: {task_name}, {estimated} {get_hour_word(estimated)}")

    return plan


def get_hour_word(hours):
    h = int(hours)
    if 11 <= h % 100 <= 14:
        return "часов"
    elif h % 10 == 1:
        return "час"
    elif 2 <= h % 10 <= 4:
        return "часа"
    else:
        return "часов"


def main():
    tasks = load_tasks()
    today = datetime.now()
    print("📋 Голосовой планировщик задач\n(скажите 'готово' для завершения)\n")

    while True:
        text = get_voice_input()
        if not text or text.lower() == "готово":
            break

        task = parse_task(text)
        if task:
            if "action" in task and task["action"] == "delete":
                name = task["task"].lower()
                before = len(tasks)
                tasks = [t for t in tasks if t.get("task", "").lower() != name]
                if len(tasks) < before:
                    print(f"✅ Задача '{name}' удалена.")
                else:
                    print(f"⚠️ Задача '{name}' не найдена.")
            else:
                if any(t["task"].lower() == task["task"].lower() for t in tasks):
                    print("⚠️ Такая задача уже есть.")
                else:
                    tasks.append(task)
                    print("✅ Задача добавлена.")
        save_tasks(tasks)

    if tasks:
        print("\n🗓 Ваш ежедневный план:")
        tasks.sort(key=lambda t: t.get("deadline") or "9999-12-31")
        plan = generate_daily_plan(tasks, today)
        for line in plan:
            print(line)
    else:
        print("📭 Задач нет.")


if __name__ == "__main__":
    main()
