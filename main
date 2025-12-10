import pygame
import random
import math

# Ініціалізація Pygame
pygame.init()

# Параметри екрану
SCREEN_WIDTH = 1100 # Збільшимо ширину для параметрів
SCREEN_HEIGHT = 700 # Трохи збільшимо висоту
INFO_PANEL_WIDTH = 250 # Збільшимо для нових параметрів
SIMULATION_AREA_WIDTH = SCREEN_WIDTH - INFO_PANEL_WIDTH

screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Трилатерація: Розширена симуляція")

# Кольори
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GREEN = (0, 128, 0)
BLUE_LIGHT = (173, 216, 230)
GREY = (200, 200, 200)
TRILATERATED_POINT_COLOR = (255, 105, 180)
LINE_COLOR = (255, 0, 0)
ORANGE = (255, 165, 0)
YELLOW_HIGHLIGHT = (255, 255, 150)
ERROR_VISUALIZATION_COLOR = (200, 200, 0, 100) # Напівпрозорий жовтий

BEACON_COLORS = [
    (0, 0, 255), (255, 0, 0), (0, 255, 0), (255, 255, 0),
    (255, 0, 255), (0, 255, 255), (255, 165, 0), (128, 0, 128)
]
next_color_index = 0

PIXELS_PER_METER = 20

#  Клас для Маяка 
class Beacon:
    def __init__(self, position_tuple, color, power_dbm, beacon_id, radius=7):
        self.position = position_tuple
        self.color = color
        self.power_dbm = power_dbm
        self.id = beacon_id
        self.radius = radius
        self.wave_timer = 0
        self.last_simulated_rssi_at_target = None
        self.last_estimated_distance_meters = None

    def get_power_mw(self):
        return 10**(self.power_dbm / 10.0)

    def __repr__(self):
        return f"Beacon(id={self.id}, pos={self.position}, A={self.power_dbm:.1f}dBm)"

# Списки та змінні
beacons = []
DEFAULT_BEACON_POWER_DBM = 15.0
MIN_BEACON_POWER_DBM = 0.0
MAX_BEACON_POWER_DBM = 23.0
POWER_STEP_DBM = 1.0
selected_beacon_index = None
next_beacon_id_char_code = ord('A')

target_point_pos = None
TARGET_POINT_RADIUS = 5

trilaterated_point_pos = None
TRILATERATED_POINT_RADIUS = 4
trilateration_circles_to_draw = []
collinearity_warning = False # Прапорець для попередження про колінеарність
last_used_beacons_for_tril = [] # Зберігаємо маяки, використані для останньої трилатерації

waves = []
BASE_WAVE_MAX_RADIUS_FROM_MW_FACTOR = 35
MIN_WAVE_RADIUS_VISUAL = 15
WAVE_GROWTH_SPEED = 0.5
WAVE_START_ALPHA = 200
WAVE_THICKNESS = 2
NEW_WAVE_INTERVAL = 45

#Параметри симуляції 
PATH_LOSS_EXPONENT_N = 3.0
RSSI_NOISE_STD_DEV_DB = 2.0
RECEIVER_SENSITIVITY_DBM = -90.0 # dBm

PATH_LOSS_N_STEP = 0.1
MIN_PATH_LOSS_N = 1.0
MAX_PATH_LOSS_N = 6.0

RSSI_NOISE_STEP = 0.1
MIN_RSSI_NOISE = 0.0
MAX_RSSI_NOISE = 10.0

# Шрифти
font = pygame.font.Font(None, 22)
small_font = pygame.font.Font(None, 18)
info_panel_font_main = pygame.font.Font(None, 20)
info_panel_font_details = pygame.font.Font(None, 18)
rssi_font = pygame.font.Font(None, 20)
distance_font = pygame.font.Font(None, 18)


#  Функції 
def calculate_distance_pixels(pos1, pos2):
    if pos1 is None or pos2 is None: return float('inf')
    return math.sqrt((pos1[0] - pos2[0])**2 + (pos1[1] - pos2[1])**2)

def simulate_rssi(beacon_power_A_dbm, distance_pixels, pixels_per_meter, path_loss_n, noise_std_dev_db):
    if pixels_per_meter <= 0: return -999.0 # Уникаємо ділення на нуль
    distance_m = distance_pixels / pixels_per_meter
    
    path_loss_db = 0.0
    if distance_m > 1.0: # A - це RSSI на відстані 1 метр
        path_loss_db = 10 * path_loss_n * math.log10(distance_m)
    elif distance_m <= 0: # Прямо на маяку або помилка
        path_loss_db = 0 # Немає втрат шляху (або максимальний сигнал)
   

    rssi_ideal = beacon_power_A_dbm - path_loss_db
    noise = 0
    if noise_std_dev_db > 0:
        noise = random.normalvariate(0, noise_std_dev_db)
    rssi_noisy = rssi_ideal + noise
    return rssi_noisy

def rssi_to_distance_meters(rssi_at_target_dbm, beacon_power_A_dbm, path_loss_n):
    # RSSI = A - 10*n*log10(d)  =>  10*n*log10(d) = A - RSSI
    # log10(d) = (A - RSSI) / (10*n)  => d = 10 ^ ((A - RSSI) / (10*n))
    if path_loss_n == 0: return float('inf')
    
    exponent_val = (beacon_power_A_dbm - rssi_at_target_dbm) / (10 * path_loss_n)
    try:
        distance_m = 10**exponent_val
    except OverflowError: 
        distance_m = float('inf') 
    return distance_m

# Функція для перевірки колінеарності трьох точок
def are_collinear(p1, p2, p3, tolerance=1e-3): # tolerance для врахування похибок float
    # Використовуємо площу трикутника (через визначник або векторний добуток).
    # Якщо площа (або її еквівалент) близька до нуля, точки колінеарні.
    # (y2 - y1)*(x3 - x2) - (y3 - y2)*(x2 - x1) == 0  (для цілих чисел)
    # Або для float: (x1(y2 - y3) + x2(y3 - y1) + x3(y1 - y2)) / 2
    area_twice = p1[0] * (p2[1] - p3[1]) + \
                 p2[0] * (p3[1] - p1[1]) + \
                 p3[0] * (p1[1] - p2[1])
    return abs(area_twice) < tolerance


def trilaterate_3_beacons(b1_pos, r1_pixels, b2_pos, r2_pixels, b3_pos, r3_pixels):
    x1, y1 = b1_pos; x2, y2 = b2_pos; x3, y3 = b3_pos
    
    # Перевірка на валідність радіусів (не повинні бути нульовими або негативними)
    if r1_pixels <=0 or r2_pixels <=0 or r3_pixels <=0:
        # print("Trilateration failed: Non-positive radius.")
        return None

    # Формули для знаходження точки перетину трьох кіл
    # (x-x1)^2 + (y-y1)^2 = r1^2
    # (x-x2)^2 + (y-y2)^2 = r2^2
    # (x-x3)^2 + (y-y3)^2 = r3^2
    # Віднімаючи перше з другого, і перше з третього, отримуємо лінійну систему.

    A = 2 * (x2 - x1)
    B = 2 * (y2 - y1)
    C = r1_pixels**2 - r2_pixels**2 + x2**2 - x1**2 + y2**2 - y1**2
    D = 2 * (x3 - x1)
    E = 2 * (y3 - y1)
    F = r1_pixels**2 - r3_pixels**2 + x3**2 - x1**2 + y3**2 - y1**2

    denominator = (A * E - D * B)
    if abs(denominator) < 1e-6: # Маяки (майже) колінеарні, або інша проблема з розв'язком
        return None

    calc_x = (C * E - F * B) / denominator
    # Потрібно обережно обрати рівняння для y, щоб уникнути ділення на нуль, якщо B або E нульові
    if abs(B) > 1e-6: # Використовуємо перше лінійне рівняння, якщо B не нуль
        calc_y = (C - A * calc_x) / B
    elif abs(E) > 1e-6: # Використовуємо друге лінійне рівняння, якщо E не нуль
        calc_y = (F - D * calc_x) / E
    else:
      
        return None
        
    return int(round(calc_x)), int(round(calc_y))

#  Головний цикл 
running = True
clock = pygame.time.Clock()

def reset_calculations():
    global trilaterated_point_pos, trilateration_circles_to_draw, collinearity_warning, last_used_beacons_for_tril
    trilaterated_point_pos = None
    trilateration_circles_to_draw.clear()
    last_used_beacons_for_tril.clear() # Очищуємо список використаних маяків
    collinearity_warning = False # Скидаємо попередження
    for beacon_obj in beacons:
        beacon_obj.last_simulated_rssi_at_target = None
        beacon_obj.last_estimated_distance_meters = None

while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT: running = False
        
        if event.type == pygame.MOUSEBUTTONDOWN:
            mouse_pos = pygame.mouse.get_pos()
            # Перевіряємо, чи клік в зоні симуляції
            if mouse_pos[0] < SIMULATION_AREA_WIDTH:
                if event.button == 1: # Ліва кнопка - Маяки
                    clicked_on_beacon_this_click = False
                    for i, beacon_obj in enumerate(beacons):
                        if calculate_distance_pixels(beacon_obj.position, mouse_pos) < beacon_obj.radius + 3: 
                            selected_beacon_index = i
                            clicked_on_beacon_this_click = True; break
                    if not clicked_on_beacon_this_click:
                        # Перевірка, чи не ставимо маяк занадто близько до існуючих
                        is_too_close = False
                        for b_obj in beacons:
                            if calculate_distance_pixels(b_obj.position, mouse_pos) < b_obj.radius * 3: # Відстань між центрами
                                is_too_close = True; break
                        if not is_too_close:
                            color = BEACON_COLORS[next_color_index % len(BEACON_COLORS)]
                            next_color_index += 1
                            current_beacon_id = chr(next_beacon_id_char_code)
                            if next_beacon_id_char_code >= ord('Z'): next_beacon_id_char_code = ord('A') 
                            else: next_beacon_id_char_code +=1
                                
                            new_beacon = Beacon(tuple(mouse_pos), color, DEFAULT_BEACON_POWER_DBM, current_beacon_id)
                            beacons.append(new_beacon)
                            selected_beacon_index = len(beacons) - 1 # Вибираємо новостворений
                            reset_calculations() # Перерахувати все
                
                elif event.button == 3: # Права кнопка - Цільова точка
                    target_point_pos = tuple(mouse_pos)
                    reset_calculations() # Перерахувати все
        
        if event.type == pygame.KEYDOWN:
            parameter_changed = False # Загальний прапорець для скидання розрахунків
            if selected_beacon_index is not None and selected_beacon_index < len(beacons):
                current_beacon = beacons[selected_beacon_index]
                if event.key == pygame.K_PLUS or event.key == pygame.K_EQUALS:
                    if current_beacon.power_dbm < MAX_BEACON_POWER_DBM:
                        current_beacon.power_dbm = round(current_beacon.power_dbm + POWER_STEP_DBM, 1)
                        parameter_changed = True
                        # Компроміс для "плавності" хвиль: скидаємо таймер, щоб нова хвиля згенерувалася швидше
                        current_beacon.wave_timer = NEW_WAVE_INTERVAL 
                elif event.key == pygame.K_MINUS:
                    if current_beacon.power_dbm > MIN_BEACON_POWER_DBM:
                        current_beacon.power_dbm = round(current_beacon.power_dbm - POWER_STEP_DBM, 1)
                        parameter_changed = True
                        current_beacon.wave_timer = NEW_WAVE_INTERVAL
                elif event.key == pygame.K_DELETE or event.key == pygame.K_BACKSPACE:
                    beacons.pop(selected_beacon_index)
                    selected_beacon_index = None # Знімаємо вибір
                    parameter_changed = True
            
            # Інтерактивне налаштування параметрів симуляції
            if event.key == pygame.K_n: # Збільшити Path Loss N
                PATH_LOSS_EXPONENT_N = min(MAX_PATH_LOSS_N, round(PATH_LOSS_EXPONENT_N + PATH_LOSS_N_STEP, 2))
                parameter_changed = True
            elif event.key == pygame.K_b: # Зменшити Path Loss N (b - "below n")
                PATH_LOSS_EXPONENT_N = max(MIN_PATH_LOSS_N, round(PATH_LOSS_EXPONENT_N - PATH_LOSS_N_STEP, 2))
                parameter_changed = True
            elif event.key == pygame.K_s: # Збільшити RSSI Noise StdDev (s - "sigma")
                RSSI_NOISE_STD_DEV_DB = min(MAX_RSSI_NOISE, round(RSSI_NOISE_STD_DEV_DB + RSSI_NOISE_STEP, 2))
                parameter_changed = True
            elif event.key == pygame.K_a: # Зменшити RSSI Noise StdDev (a - "above s" :) )
                RSSI_NOISE_STD_DEV_DB = max(MIN_RSSI_NOISE, round(RSSI_NOISE_STD_DEV_DB - RSSI_NOISE_STEP, 2))
                parameter_changed = True

            if parameter_changed: reset_calculations()
            
            if event.key == pygame.K_ESCAPE: selected_beacon_index = None
            elif event.key == pygame.K_c: # Очистити цільову точку та розрахунки
                target_point_pos = None; reset_calculations()

    active_waves_next_frame = []
    for beacon_obj in beacons:
        beacon_obj.wave_timer += 1
        if beacon_obj.wave_timer >= NEW_WAVE_INTERVAL:
            power_mw = beacon_obj.get_power_mw()
            calculated_max_rad = (power_mw**0.5) * BASE_WAVE_MAX_RADIUS_FROM_MW_FACTOR
            b_max_wave_radius = max(MIN_WAVE_RADIUS_VISUAL, calculated_max_rad)
            waves.append([beacon_obj.position[0], beacon_obj.position[1], 0, # cx, cy, current_radius
                          b_max_wave_radius, WAVE_GROWTH_SPEED, WAVE_START_ALPHA, beacon_obj.color])
            beacon_obj.wave_timer = 0 # Скидаємо таймер
    
    for wave in waves: # cx, cy, rad, max_rad, speed, alpha, color
        wave[2] += wave[4] # rad += speed
        if wave[3] > 0: # max_rad
            current_alpha_ratio = (1 - (wave[2] / wave[3])**1.5) # rad / max_rad
            wave[5] = max(0, int(WAVE_START_ALPHA * current_alpha_ratio)) # alpha
        else:
            wave[5] = 0 # alpha
        
        if wave[5] > 5 and wave[2] < wave[3] + wave[4]: # alpha > 5 and rad < max_rad + speed
            active_waves_next_frame.append(wave)
    waves = active_waves_next_frame


    #  Розрахунок RSSI, Відстаней та Трилатерація 
    if target_point_pos:
        beacons_with_valid_data_for_tril = []
        for beacon_obj in beacons:
            # Розрахунок RSSI (якщо ще не було)
            if beacon_obj.last_simulated_rssi_at_target is None: 
                dist_pixels = calculate_distance_pixels(beacon_obj.position, target_point_pos)
                beacon_obj.last_simulated_rssi_at_target = simulate_rssi(
                    beacon_obj.power_dbm, dist_pixels, PIXELS_PER_METER,
                    PATH_LOSS_EXPONENT_N, RSSI_NOISE_STD_DEV_DB)

            # Розрахунок відстані за RSSI (якщо ще не було і RSSI дозволяє)
            if beacon_obj.last_estimated_distance_meters is None and \
               beacon_obj.last_simulated_rssi_at_target is not None:
                if beacon_obj.last_simulated_rssi_at_target >= RECEIVER_SENSITIVITY_DBM:
                    beacon_obj.last_estimated_distance_meters = rssi_to_distance_meters(
                        beacon_obj.last_simulated_rssi_at_target, beacon_obj.power_dbm, PATH_LOSS_EXPONENT_N)
                else:
                    beacon_obj.last_estimated_distance_meters = float('inf') # Сигнал занадто слабкий

            # Додаємо маяк до списку для трилатерації, якщо є валідна розрахункова відстань
            if beacon_obj.last_estimated_distance_meters is not None and \
               not math.isinf(beacon_obj.last_estimated_distance_meters) and \
               beacon_obj.last_estimated_distance_meters > 0: # Радіус має бути позитивним
                dist_pixels_est = beacon_obj.last_estimated_distance_meters * PIXELS_PER_METER
                beacons_with_valid_data_for_tril.append({
                    "pos": beacon_obj.position, "dist_pixels": dist_pixels_est, 
                    "color": beacon_obj.color, "id": beacon_obj.id,
                    "original_beacon_obj": beacon_obj # Зберігаємо посилання на об'єкт маяка
                })
        
        # Скидаємо попередні результати трилатерації перед новим розрахунком
        trilaterated_point_pos = None 
        trilateration_circles_to_draw.clear()
        last_used_beacons_for_tril.clear()
        collinearity_warning = False 

        if len(beacons_with_valid_data_for_tril) >= 3:
            # Сортуємо маяки за розрахунковою відстанню (від меншої до більшої)
            beacons_with_valid_data_for_tril.sort(key=lambda b: b["dist_pixels"])
            
            # Беремо перші три маяки для трилатерації
            b1_data = beacons_with_valid_data_for_tril[0]
            b2_data = beacons_with_valid_data_for_tril[1]
            b3_data = beacons_with_valid_data_for_tril[2]
            
            # Зберігаємо маяки, які будуть використані
            last_used_beacons_for_tril = [
                b1_data["original_beacon_obj"], 
                b2_data["original_beacon_obj"], 
                b3_data["original_beacon_obj"]
            ]
            
            # Перевірка на колінеарність трьох обраних маяків
            if are_collinear(b1_data["pos"], b2_data["pos"], b3_data["pos"]):
                collinearity_warning = True
                calculated_pos = None # Не намагаємося трилатерувати, якщо колінеарні
            else:
                collinearity_warning = False # Скидаємо, якщо були попередження раніше
                calculated_pos = trilaterate_3_beacons(
                    b1_data["pos"], b1_data["dist_pixels"],
                    b2_data["pos"], b2_data["dist_pixels"],
                    b3_data["pos"], b3_data["dist_pixels"]
                )
            
            trilaterated_point_pos = calculated_pos # Оновлюємо розраховану позицію (може бути None)
            if calculated_pos: # Якщо точка розрахована успішно
                # Готуємо кола для візуалізації
                trilateration_circles_to_draw = [
                    (b1_data["pos"], b1_data["dist_pixels"], b1_data["color"]),
                    (b2_data["pos"], b2_data["dist_pixels"], b2_data["color"]),
                    (b3_data["pos"], b3_data["dist_pixels"], b3_data["color"])
                ]
            

    #  Рендеринг 
    # Зона симуляції
    screen.fill(WHITE, (0, 0, SIMULATION_AREA_WIDTH, SCREEN_HEIGHT))
    # Інфо-панель (фон)
    pygame.draw.rect(screen, GREY, (SIMULATION_AREA_WIDTH, 0, INFO_PANEL_WIDTH, SCREEN_HEIGHT))
    pygame.draw.line(screen, BLACK, (SIMULATION_AREA_WIDTH, 0), (SIMULATION_AREA_WIDTH, SCREEN_HEIGHT), 2)


    # Хвилі
    for wave_data in waves: 
        cx, cy, rad, max_rad, speed, alpha, color = wave_data
        if cx > SIMULATION_AREA_WIDTH + max_rad : continue # Не малювати хвилі, що повністю за панеллю
        if rad > 0 and alpha > 0:
            surface_size = int(rad * 2); 
            if surface_size <= 0: continue # Пропускаємо, якщо радіус занадто малий
            temp_surface = pygame.Surface((surface_size, surface_size), pygame.SRCALPHA)
            try: # Захист від помилок малювання (наприклад, якщо радіус дуже великий)
                pygame.draw.circle(temp_surface, (*color, alpha), (int(rad), int(rad)), int(rad), WAVE_THICKNESS); 
                screen.blit(temp_surface, (cx - rad, cy - rad))
            except pygame.error: 
                pass # Ігноруємо помилку і продовжуємо

    # Маяки, ID, лінії відстаней
    for i, beacon_obj in enumerate(beacons):
        draw_color = beacon_obj.color; outline_color = BLACK
        # Підсвітка маяків, що використовуються для поточної трилатерації
        is_used_for_tril = beacon_obj in last_used_beacons_for_tril 
        
        if i == selected_beacon_index:
            pygame.draw.circle(screen, RED, beacon_obj.position, beacon_obj.radius + 4, 3) 
            outline_color = RED # ID теж буде червоним
        elif is_used_for_tril and target_point_pos: # Підсвітка, якщо використовується
             pygame.draw.circle(screen, ORANGE, beacon_obj.position, beacon_obj.radius + 2, 2)


        pygame.draw.circle(screen, draw_color, beacon_obj.position, beacon_obj.radius)
        pygame.draw.circle(screen, outline_color, beacon_obj.position, beacon_obj.radius, 1)
        
        id_text_surface = font.render(str(beacon_obj.id), True, outline_color if i == selected_beacon_index else BLACK)
        id_text_rect = id_text_surface.get_rect(center=(beacon_obj.position[0], beacon_obj.position[1] - beacon_obj.radius - 10))
        screen.blit(id_text_surface, id_text_rect)

        # Лінії та текст відстані від маяка до СПРАВЖНЬОЇ цілі (на основі РОЗРАХУНКОВОЇ відстані)
        if target_point_pos and beacon_obj.last_estimated_distance_meters is not None and \
           not math.isinf(beacon_obj.last_estimated_distance_meters) and \
           beacon_obj.last_simulated_rssi_at_target >= RECEIVER_SENSITIVITY_DBM : # Малюємо лінію тільки якщо сигнал "чутно"
            pygame.draw.line(screen, LINE_COLOR, beacon_obj.position, target_point_pos, 1) # Тонша лінія
            
            # Текст з розрахунковою відстанню
            mid_x = (beacon_obj.position[0] + target_point_pos[0]) / 2
            mid_y = (beacon_obj.position[1] + target_point_pos[1]) / 2
            dist_text = f"{beacon_obj.last_estimated_distance_meters:.1f}m"
            dist_surface = distance_font.render(dist_text, True, BLACK, GREY) # Додав фон для кращої читабельності
            dist_rect = dist_surface.get_rect(center=(mid_x, mid_y - 10))
            screen.blit(dist_surface, dist_rect)
            
    # Візуалізація кіл трилатерації (на основі розрахункових відстаней)
    if target_point_pos and trilateration_circles_to_draw:
        for center_pos, radius_pixels, b_color in trilateration_circles_to_draw:
            if radius_pixels > 0:
                circle_surface_size = int(radius_pixels * 2)
                if circle_surface_size <=0: continue
                temp_circle_surface = pygame.Surface((circle_surface_size, circle_surface_size), pygame.SRCALPHA)
                try:
                    r, g, b_val = b_color 
                    alpha_for_tril_circle = 60 # Трохи яскравіше
                    pygame.draw.circle(temp_circle_surface, (r,g,b_val, alpha_for_tril_circle), 
                                       (int(radius_pixels), int(radius_pixels)), int(radius_pixels), 2) # Товщина кола 2
                    screen.blit(temp_circle_surface, (center_pos[0] - radius_pixels, center_pos[1] - radius_pixels))
                except pygame.error: pass

    # Цільова точка (справжня)
    if target_point_pos:
        pygame.draw.circle(screen, GREEN, target_point_pos, TARGET_POINT_RADIUS)
        pygame.draw.circle(screen, BLACK, target_point_pos, TARGET_POINT_RADIUS, 1)

    # Розрахована трилатерацією точка
    if trilaterated_point_pos:
        pygame.draw.circle(screen, TRILATERATED_POINT_COLOR, trilaterated_point_pos, TRILATERATED_POINT_RADIUS)
        pygame.draw.circle(screen, BLACK, trilaterated_point_pos, TRILATERATED_POINT_RADIUS, 1)
        
        # Візуалізація похибки (лінія між справжньою та розрахованою ціллю)
        if target_point_pos:
            pygame.draw.line(screen, RED, target_point_pos, trilaterated_point_pos, 1) # Тонка червона лінія
            
            # Візуалізація похибки (просте коло невизначеності)
            error_dist_pixels = calculate_distance_pixels(target_point_pos, trilaterated_point_pos)
            if error_dist_pixels > 0 and error_dist_pixels < SIMULATION_AREA_WIDTH / 2 : # Обмеження на розмір, щоб не заповнювати все
                error_surface_size = int(error_dist_pixels * 2)
                if error_surface_size > 0:
                    temp_error_surface = pygame.Surface((error_surface_size, error_surface_size), pygame.SRCALPHA)
                    try:
                        pygame.draw.circle(temp_error_surface, ERROR_VISUALIZATION_COLOR, # Напівпрозорий колір
                                           (int(error_dist_pixels), int(error_dist_pixels)), int(error_dist_pixels), 0) # Заповнене коло
                        screen.blit(temp_error_surface, (trilaterated_point_pos[0] - error_dist_pixels, trilaterated_point_pos[1] - error_dist_pixels))
                    except pygame.error: pass


    #  Інформаційна панель 
    INFO_PANEL_X_START = SIMULATION_AREA_WIDTH 
    pygame.draw.rect(screen, GREY, (INFO_PANEL_X_START, 0, INFO_PANEL_WIDTH, SCREEN_HEIGHT)) # Перемальовуємо фон панелі
    pygame.draw.line(screen, BLACK, (INFO_PANEL_X_START, 0), (INFO_PANEL_X_START, SCREEN_HEIGHT), 2)

    info_y_pos = 10
    line_height = 18 # Трохи збільшив для кращої читабельності
    panel_text_margin = SIMULATION_AREA_WIDTH + 10

    def draw_text_on_panel(text, y_pos, color=BLACK, font_to_use=info_panel_font_details, bold=False, highlight_bg_color=None):
        current_font = font_to_use
        if bold: current_font.set_bold(True)
        
        text_surface = current_font.render(text, True, color)
        
        if highlight_bg_color: # Підсвічування для вибраного маяка або важливої інформації
            highlight_rect = pygame.Rect(panel_text_margin - 3, y_pos -1, INFO_PANEL_WIDTH - 10 , line_height -1)
            pygame.draw.rect(screen, highlight_bg_color, highlight_rect)

        screen.blit(text_surface, (panel_text_margin, y_pos))
        
        if bold: current_font.set_bold(False) # Скидаємо жирність для наступних викликів
        return y_pos + line_height
    
    # Загаловок панелі
    info_y_pos = draw_text_on_panel("Інформація:", info_y_pos, font_to_use=info_panel_font_main, bold=True)
    info_y_pos += 5

    # Інформація про всі маяки
    if not beacons:
        info_y_pos = draw_text_on_panel("Маяки не розміщено.", info_y_pos)
    else:
        info_y_pos = draw_text_on_panel(f"Маяків: {len(beacons)}", info_y_pos, bold=True)
        for i, beacon_obj in enumerate(beacons):
            is_selected = (i == selected_beacon_index)
            bg_color = YELLOW_HIGHLIGHT if is_selected else None
            prefix = "-> " if is_selected else "   " # Позначка для вибраного
            
            # Основна інформація про маяк в один рядок
            beacon_info_line1 = f"{prefix}ID: {beacon_obj.id} ({beacon_obj.position[0]},{beacon_obj.position[1]}) A:{beacon_obj.power_dbm:.1f}dBm"
            info_y_pos = draw_text_on_panel(beacon_info_line1, info_y_pos, color=beacon_obj.color, bold=is_selected, highlight_bg_color=bg_color)
            
            # Додаткова інформація, якщо є ціль
            if target_point_pos:
                rssi_text = ""
                dist_text = ""
                if beacon_obj.last_simulated_rssi_at_target is not None:
                    rssi_text = f"{beacon_obj.last_simulated_rssi_at_target:.1f}dBm"
                    if beacon_obj.last_simulated_rssi_at_target < RECEIVER_SENSITIVITY_DBM: rssi_text += " (дуже слабко)"
                
                if beacon_obj.last_estimated_distance_meters is not None:
                    if math.isinf(beacon_obj.last_estimated_distance_meters): dist_text = "> діапазон"
                    else: dist_text = f"{beacon_obj.last_estimated_distance_meters:.1f}m"
                
                info_y_pos = draw_text_on_panel(f"     До цілі: RSSI {rssi_text}, Розрах.відст. {dist_text}", info_y_pos, font_to_use=small_font)
            info_y_pos += 2 # Невеликий відступ між маяками
    info_y_pos += line_height / 2 # Відступ після списку маяків

    # Розділювач
    pygame.draw.line(screen, BLACK, (panel_text_margin - 5, info_y_pos), (SCREEN_WIDTH - 5, info_y_pos), 1)
    info_y_pos += 5

    # Інформація про цільову точку
    if target_point_pos:
        info_y_pos = draw_text_on_panel(f"Ціль (справжня): ({target_point_pos[0]}, {target_point_pos[1]})", info_y_pos, color=GREEN, bold=True)
    else:
        info_y_pos = draw_text_on_panel("Ціль не задано.", info_y_pos, color=GREEN, bold=True)
    info_y_pos += line_height / 2

    # Інформація про результат трилатерації
    if trilaterated_point_pos:
        info_y_pos = draw_text_on_panel(f"Трилатерація: ({trilaterated_point_pos[0]}, {trilaterated_point_pos[1]})", info_y_pos, color=TRILATERATED_POINT_COLOR, bold=True)
        # Відображення ID маяків, використаних для трилатерації
        ids_used = ", ".join([b.id for b in last_used_beacons_for_tril]) if last_used_beacons_for_tril else "Н/Д"
        info_y_pos = draw_text_on_panel(f"  Маяки: {ids_used}", info_y_pos, font_to_use=small_font)
        if target_point_pos:
            error_dist_pixels = calculate_distance_pixels(target_point_pos, trilaterated_point_pos)
            error_dist_meters = error_dist_pixels / PIXELS_PER_METER
            info_y_pos = draw_text_on_panel(f"  Помилка: {error_dist_meters:.2f} m ({error_dist_pixels:.1f} px)", info_y_pos, color=RED)
    elif target_point_pos and len(beacons_with_valid_data_for_tril) <3 and len(beacons) >=3: # Якщо є ціль, є >3 маяки, але <3 з валідними даними
         info_y_pos = draw_text_on_panel("Трилатерація: недостатньо даних від маяків (сигнал, тощо)", info_y_pos, color=ORANGE, bold=True)
    elif target_point_pos and len(beacons) < 3: # Якщо є ціль, але менше 3 маяків взагалі
         info_y_pos = draw_text_on_panel("Трилатерація: потрібно мін. 3 маяки", info_y_pos, color=ORANGE, bold=True)
    else: # Якщо немає цілі, або ще якісь умови
         info_y_pos = draw_text_on_panel("Трилатерація: немає рез.", info_y_pos, color=TRILATERATED_POINT_COLOR, bold=True)

    # Попередження про колінеарність
    if collinearity_warning:
        info_y_pos = draw_text_on_panel("УВАГА: Маяки колінеарні!", info_y_pos, color=RED, bold=True, highlight_bg_color=(255,200,200))
    info_y_pos += line_height

    # Розділювач
    pygame.draw.line(screen, BLACK, (panel_text_margin - 5, info_y_pos), (SCREEN_WIDTH - 5, info_y_pos), 1)
    info_y_pos += 5

    # Параметри симуляції
    info_y_pos = draw_text_on_panel("Параметри симуляції:", info_y_pos, bold=True)
    info_y_pos = draw_text_on_panel(f"  Масштаб: {PIXELS_PER_METER} px/m", info_y_pos)
    info_y_pos = draw_text_on_panel(f"  Загасання n (N/B): {PATH_LOSS_EXPONENT_N:.1f}", info_y_pos)
    info_y_pos = draw_text_on_panel(f"  Шум RSSI σ (S/A): {RSSI_NOISE_STD_DEV_DB:.1f}dB", info_y_pos)
    info_y_pos = draw_text_on_panel(f"  Чутливість Rx: {RECEIVER_SENSITIVITY_DBM:.0f}dBm", info_y_pos)
    info_y_pos += line_height


    # Інструкції (зліва, на основній області)
    instr_y_start = 10
    instr_line_height = 20
    instructions = [
        "ЛКМ: Маяк (вибір/нов.)", "ПКМ: Цільова точка",
        "DEL/Backspace: Видал. маяк", "+/- : Потужність A (dBm)",
        "ESC: Зняти вибір", "C: Очистити ціль",
        "N/B: Змінити 'n' (загасання)",
        "S/A: Змінити 'σ' (шум RSSI)"
    ]
    for line in instructions:
        text_surface = font.render(line, True, BLACK)
        screen.blit(text_surface, (10, instr_y_start))
        instr_y_start += instr_line_height


    pygame.display.flip()
    clock.tick(60)


pygame.quit()
