import logging
from bs4 import BeautifulSoup
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes
import os
import re
from typing import List, Tuple
import chardet
import requests

# Включаем логирование
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# Токен бота
TELEGRAM_TOKEN = os.getenv('TG_TOKEN')

# URL для получения расписания
SCHEDULE_URL = 'https://ppkslavyanova.ru/lessonlist'

# Группы и их индексы
GROUPS = {
    "Первый курс": ["КС-24/1, КС-24/1к", "КС-24/2, КС-24/2к", "КС-24/3к", "СА-24/1, СА-24/1к", "СА-24/2к", "СА-24/3к",
                    "Т-24/1, Т-24/1к", "Т-24/2, Т-24/2к", "Т-24/3к", "ТД-24/1, ТД-24/1к", "ТД-24/2к",
                    "ТМ-24/1, ТМ-24/1к", "ТМ-24/2, ТМ-24/2к", "ТМ-24/3, ТМ-24/3к", "ТМ-24/4к", "УМ-24/1, УМ-24/1к",
                    "УМ-24/2, УМ-24/2к", "УП-24/1, УП-24/1к", "УП-24/2, УП-24/2к", "Э-24, Э-24к"],
    "Второй курс": ["КС-23/1, КС-23/1к", "КС-23/2, КС-23/2к", "КС-23/3к", "СА-23/1, СА-23/1к", "СА-23/2к", "СА-23/3к",
                    "Т-23/1, Т-23/1к", "Т-23/2, Т-23/2к", "Т-23/3к", "ТД-23/1, ТД-23/1к", "ТД-23/2к",
                    "ТМ-23/1, ТМ-23/1к", "ТМ-23/2, ТМ-23/2к", "УМ-23/1, УМ-23/1к", "УМ-23/2, УМ-23/2к",
                    "УП-23/1, УП-23/1к", "УП-23/2, УП-23/2к", "Эл-23, Эл-23к"],
    "Третий курс": ["КС-22/1, КС-22/1к", "КС-22/2, КС-22/2к", "КС-22/3к", "СА-22", "СА-22к", "Св-22, Св-22к",
                    "Т-22/1, Т-22/1к", "Т-22/2, Т-22/2к", "ТД-22к", "ТМ-22/1, ТМ-22/1к", "ТМ-22/2, ТМ-22/2к",
                    "УМ-22, УМ-22к", "УП-22/1, УП-22/1к", "УП-22/2, УП-22/2к", "Эл-22, Эл-22к"],
    "Четвертый курс": ["КС-21/1", "КС-21/2", "КС-21/3к", "СА-21", "СА-21к", "Т-21/1, Т-21/1к", "Т-21/2, Т-21/2к",
                       "ТД-21к", "ТМ-21/1, ТМ-21/1к", "ТМ-21/2, ТМ-21/2к", "УМ-21, УМ-21к"]
}

# Словарь, сопоставляющий названия групп с их индексами
GROUP_INDEX_MAPPING = {
    "КС-24/1, КС-24/1к": "index0",
    "КС-24/2, КС-24/2к": "index1",
    "КС-24/3к": "index2",
    "СА-24/1, СА-24/1к": "index3",
    "СА-24/2к": "index4",
    "СА-24/3к": "index5",
    "Т-24/1, Т-24/1к": "index6",
    "Т-24/2, Т-24/2к": "index7",
    "Т-24/3к": "index8",
    "ТД-24/1, ТД-24/1к": "index9",
    "ТД-24/2к": "index10",
    "ТМ-24/1, ТМ-24/1к": "index11",
    "ТМ-24/2, ТМ-24/2к": "index12",
    "ТМ-24/3, ТМ-24/3к": "index13",
    "ТМ-24/4к": "index14",
    "УМ-24/1, УМ-24/1к": "index15",
    "УМ-24/2, УМ-24/2к": "index16",
    "УП-24/1, УП-24/1к": "index17",
    "УП-24/2, УП-24/2к": "index18",
    "Э-24, Э-24к": "index19",

    "КС-23/1, КС-23/1к": "index20",
    "КС-23/2, КС-23/2к": "index21",
    "КС-23/3к": "index22",
    "СА-23/1, СА-23/1к": "index23",
    "СА-23/2к": "index24",
    "СА-23/3к": "index25",
    "Т-23/1, Т-23/1к": "index26",
    "Т-23/2, Т-23/2к": "index27",
    "Т-23/3к": "index28",
    "ТД-23/1, ТД-23/1к": "index29",
    "ТД-23/2к": "index30",
    "ТМ-23/1, ТМ-23/1к": "index31",
    "ТМ-23/2, ТМ-23/2к": "index32",
    "УМ-23/1, УМ-23/1к": "index33",
    "УМ-23/2, УМ-23/2к": "index34",
    "УП-23/1, УП-23/1к": "index35",
    "УП-23/2, УП-23/2к": "index36",
    "Эл-23, Эл-23к": "index37",

    "КС-22/1, КС-22/1к": "index38",
    "КС-22/2, КС-22/2к": "index39",
    "КС-22/3к": "index40",
    "СА-22": "index41",
    "СА-22к": "index42",
    "Св-22, Св-22к": "index43",
    "Т-22/1, Т-22/1к": "index44",
    "Т-22/2, Т-22/2к": "index45",
    "ТД-22к": "index46",
    "ТМ-22/1, ТМ-22/1к": "index47",
    "ТМ-22/2, ТМ-22/2к": "index48",
    "УМ-22, УМ-22к": "index49",
    "УП-22/1, УП-22/1к": "index50",
    "УП-22/2, УП-22/2к": "index51",
    "Эл-22, Эл-22к": "index52",

    "КС-21/1": "index53",
    "КС-21/2": "index54",
    "КС-21/3к": "index55",
    "СА-21": "index56",
    "СА-21к": "index57",
    "Т-21/1, Т-21/1к": "index58",
    "Т-21/2, Т-21/2к": "index59",
    "ТД-21к": "index60",
    "ТМ-21/1, ТМ-21/1к": "index61",
    "ТМ-21/2, ТМ-21/2к": "index62",
    "УМ-21, УМ-21к": "index63",
    # При необходимости можно добавить больше индексов
}


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Отправляет приветственное сообщение с кнопками групп."""
    keyboard = [[InlineKeyboardButton(course, callback_data=course)] for course in GROUPS.keys()]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text('Выберите курс:', reply_markup=reply_markup)


async def show_groups(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Показывает группы для выбранного курса."""
    query = update.callback_query
    await query.answer()

    course = query.data
    keyboard = [[InlineKeyboardButton(group, callback_data=group)] for group in GROUPS[course]]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await query.edit_message_text(text=f'Выберите группу для {course}:', reply_markup=reply_markup)


async def fetch_dates() -> List[Tuple[str, str]]:
    """Получает доступные даты с сайта."""
    try:
        headers = {'User-Agent': 'Mozilla/5.0 ...'}
        response = requests.get(SCHEDULE_URL, headers=headers)
        response.raise_for_status()
        logging.info("Запрос успешен, получили ответ от сервера.")

        response.encoding = 'utf-8'
        soup = BeautifulSoup(response.text, 'html.parser')
        date_links = soup.select('div.lesson_list a.link')

        if not date_links:
            logging.warning("Ссылок с классом 'link' не найдено в HTML.")
            return []

        dates = []
        for link in date_links:
            date_text = link.text.strip()
            date_href = link['href'].strip()

            if is_valid_date_format(date_text):
                dates.append((date_text, date_href))  # Сохраняем текст и ссылку на дату
                logging.info(f"Добавлена дата: {date_text}, ссылка: {date_href}")

        logging.info(f"Всего найдено дат: {len(dates)}")
        return dates
    except requests.RequestException as e:
        logging.error(f"Ошибка при получении дат: {e}")
        return []


def is_valid_date_format(date_string: str) -> bool:
    """Проверяет, соответствует ли строка формату даты."""
    date_pattern = r'^\d{1,2} [а-яА-ЯёЁ]+\s*$'  # Формат "дд месяц"
    return re.match(date_pattern, date_string) is not None


async def request_date(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Запрашивает дату после выбора группы."""
    query = update.callback_query
    await query.answer()

    group = query.data  # Получаем выбранную группу
    context.user_data['selected_group'] = group  # Сохраняем группу в контексте пользователя
    dates = await fetch_dates()  # Получаем доступные даты

    if dates:
        keyboard = [[InlineKeyboardButton(date[0], callback_data=date[1])] for date in dates]  # date[0] — текст, date[1] — ссылка
        reply_markup = InlineKeyboardMarkup(keyboard)
        await query.edit_message_text(text=f'Выберите дату для группы {group}:', reply_markup=reply_markup)
    else:
        await query.edit_message_text(text='Ошибка получения доступных дат для группы.')


async def button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обрабатывает нажатия на кнопки групп и дат."""
    query = update.callback_query
    await query.answer()  # Подтверждаем нажатие кнопки

    if query.data in GROUPS:  # Это была кнопка с курсом
        await show_groups(update, context)
    elif query.data in GROUP_INDEX_MAPPING:  # Это была кнопка с группой
        await request_date(update, context)
    else:  # Это кнопка с датой
        selected_group = context.user_data.get('selected_group')

        if not selected_group:
            await query.edit_message_text(text="Ошибка: группа не выбрана. Пожалуйста, начните заново.")
            return

        # Получаем день, его параметр и текст даты
        selected_date = query.data  # Это параметр, полученный из fetch_dates

        # Здесь нужно получить текст даты из dates
        # Это может потребовать модификации кода получения списка дат

        dates = await fetch_dates()  # Получаем доступные даты
        date_text = next((date[0] for date in dates if date[1] == selected_date), None)

        if date_text is None:
            await query.edit_message_text(text="Ошибка: не удалось получить текст даты.")
            return

        # Получаем расписание, передавая правильные аргументы
        schedule = await fetch_schedule(selected_group, selected_date, date_text)
        await query.edit_message_text(text=schedule)


async def fetch_schedule(group: str, day_param: str, selected_date: str) -> str:
    """Получает расписание для группы из URL."""
    try:
        full_url = SCHEDULE_URL + day_param  # Формируем полный URL для запроса
        response = requests.get(full_url)
        response.raise_for_status()

        # Определяем кодировку
        detected_encoding = chardet.detect(response.content)
        response.encoding = detected_encoding['encoding'] if detected_encoding['confidence'] > 0.5 else 'utf-8'

        soup = BeautifulSoup(response.text, 'html.parser')

        # Находим блок с расписанием
        lesson_list = soup.find('ul', class_='lesson_on_group')

        if not lesson_list:
            return "Расписание не найдено на странице."

        # Извлекаем строки таблицы
        lesson_items = lesson_list.find_all('li', class_='lesson_cart')

        group_index = GROUP_INDEX_MAPPING.get(group)

        if not group_index:
            return f"Группа {group} не найдена в списке."

        schedule = []
        for item in lesson_items:
            if item.get('id') == group_index:
                discipline_items = item.find_all('div', class_='discepline_item')
                for discipline in discipline_items:
                    # Пробуем извлечь индекс дисциплины
                    index_element = discipline.find('div', class_='index inline')
                    index = index_element.text.strip() if index_element else 'N/A'

                    # Извлекаем название дисциплины
                    name_element = discipline.find('div', class_='discipline_name inline')
                    name = name_element.text.strip() if name_element else 'N/A'

                    if 'начало:' in name:
                        name = name.split('начало:')[0].strip()

                    # Извлекаем время
                    time_start_element = name_element.find('span') if name_element else None
                    if time_start_element:
                        raw_time = time_start_element.text.strip().replace('<sup>', '').replace('</sup>', '').strip()

                        if 'начало:' in raw_time:
                            raw_time = raw_time.split('начало:')[1].strip()

                        if len(raw_time) == 4:  # Например, '0830'
                            hours = int(raw_time[:2])  # Первые 2 символа
                            minutes = int(raw_time[2:])  # Последние 2 символа
                            time_start = f"{hours:02}:{minutes:02}"
                        else:
                            time_start = 'N/A'  # Если формат некорректный
                    else:
                        time_start = 'N/A'

                    # Извлекаем преподавателя
                    teacher = discipline.find('div', class_='teacher').text.strip() if discipline.find('div',
                                                                                                       class_='teacher') else 'N/A'

                    # Извлекаем место и класс, добавлена обработка для территории
                    location = discipline.find('div', 'location inline').text.strip() if discipline.find('div',
                                                                                                         'location inline') else 'N/A'
                    location_string = discipline.find('div', 'location_string').text.strip() if discipline.find('div',
                                                                                                                'location_string') else 'N/A'

                    # Если определено значение из location_string, используем его
                    if location_string and location_string != 'N/A':
                        location = location_string

                    classroom = discipline.find('div', 'classroom inline').text.strip() if discipline.find('div',
                                                                                                           'classroom inline') else 'N/A'

                    # Формируем строку расписания
                    entry = f"{name} ({time_start}"

                    if teacher and teacher != 'N/A':
                        entry += f", преподаватель: {teacher}"

                    if location and location != 'N/A':
                        entry += f", место: {location}"

                    if classroom and classroom != 'N/A':
                        entry += f", класс: {classroom}"

                    entry += ")"  # Закрываем скобку

                    if index and index != 'N/A':
                        entry = f"{index}. {entry}"

                    schedule.append(entry)

        if not schedule:
            return f"Расписание для группы {group} не найдено."

        # Возвращаем расписание с выбранной датой
        return f"Расписание для {group} на {selected_date}:\n" + "\n".join(schedule)

    except requests.RequestException as e:
        logging.error(f"Ошибка при получении расписания: {e}")
        return "Не удалось получить расписание. Пожалуйста, попробуйте позже."

def main() -> None:
    """Запуск бота."""
    application = ApplicationBuilder().token(TELEGRAM_TOKEN).build()

    # Регистрация обработчиков команд и кнопок
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button))

    application.run_polling()


if __name__ == '__main__':
    main()
