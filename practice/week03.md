import pygame
import sys
import math
import random

pygame.init()
screen = pygame.display.set_mode((800, 600))
pygame.display.set_caption("My First Pygame")

WIDTH = 800
HEIGHT = 600

WHITE = (255, 255, 255)
BLUE = (0, 0, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)

clock = pygame.time.Clock()
font = pygame.font.SysFont(None, 30)

running = True

x = 400
y = 300
radius = 50

speed = 300
wall_thickness = 10

start_time = pygame.time.get_ticks()
score = 0

# 위험구역 관련
warning_length = 40
warning_interval = 2000
last_warning_time = 0
min_gap = 20
max_warnings = 8
warning_delay = 1000  # 1초 뒤 발사

warnings = []   # 현재 경고 구역
bullets = []    # 발사된 삼각형 장애물

bullet_speed = 350
bullet_size = 18


def is_too_close(side, pos, warnings, warning_length, min_gap):
    for warning in warnings:
        existing_side = warning["side"]
        existing_pos = warning["pos"]

        if side == existing_side:
            if pos < existing_pos + warning_length + min_gap and existing_pos < pos + warning_length + min_gap:
                return True
    return False


def make_warning(side, pos, created_time):
    return {
        "side": side,
        "pos": pos,
        "created_time": created_time,
        "shot": False
    }


def make_bullet(side, pos):
    if side == "left":
        return {
            "side": side,
            "x": wall_thickness + bullet_size,
            "y": pos + warning_length / 2,
            "vx": bullet_speed,
            "vy": 0
        }
    elif side == "right":
        return {
            "side": side,
            "x": WIDTH - wall_thickness - bullet_size,
            "y": pos + warning_length / 2,
            "vx": -bullet_speed,
            "vy": 0
        }
    elif side == "top":
        return {
            "side": side,
            "x": pos + warning_length / 2,
            "y": wall_thickness + bullet_size,
            "vx": 0,
            "vy": bullet_speed
        }
    elif side == "bottom":
        return {
            "side": side,
            "x": pos + warning_length / 2,
            "y": HEIGHT - wall_thickness - bullet_size,
            "vx": 0,
            "vy": -bullet_speed
        }


def get_triangle_points(bullet):
    x = bullet["x"]
    y = bullet["y"]
    s = bullet_size

    if bullet["side"] == "left":
        return [(x + s, y), (x - s, y - s), (x - s, y + s)]
    elif bullet["side"] == "right":
        return [(x - s, y), (x + s, y - s), (x + s, y + s)]
    elif bullet["side"] == "top":
        return [(x, y + s), (x - s, y - s), (x + s, y - s)]
    elif bullet["side"] == "bottom":
        return [(x, y - s), (x - s, y + s), (x + s, y + s)]


while running:
    dt = clock.tick(60) / 1000
    current_ticks = pygame.time.get_ticks()

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    keys = pygame.key.get_pressed()

    move_x = 0
    move_y = 0

    # 방향키 입력
    if keys[pygame.K_LEFT]:
        move_x -= 1
    if keys[pygame.K_RIGHT]:
        move_x += 1
    if keys[pygame.K_UP]:
        move_y -= 1
    if keys[pygame.K_DOWN]:
        move_y += 1

    # 쉬프트 누르면 속도 2배
    current_speed = speed
    if keys[pygame.K_LSHIFT] or keys[pygame.K_RSHIFT]:
        current_speed = speed * 2

    # 대각선 속도 보정
    if move_x != 0 and move_y != 0:
        move_x /= math.sqrt(2)
        move_y /= math.sqrt(2)

    # 이동
    x += move_x * current_speed * dt
    y += move_y * current_speed * dt

    # 벽 안쪽으로만 움직이게 제한
    if x - radius < wall_thickness:
        x = wall_thickness + radius
    if x + radius > WIDTH - wall_thickness:
        x = WIDTH - wall_thickness - radius
    if y - radius < wall_thickness:
        y = wall_thickness + radius
    if y + radius > HEIGHT - wall_thickness:
        y = HEIGHT - wall_thickness - radius

    # 경과 시간
    elapsed_time = (current_ticks - start_time) / 1000

    # 점수 증가
    score_speed = 5 + elapsed_time * 0.8
    score += score_speed * dt

    # 30초마다 위험구역 개수 증가, 최대 8개
    warning_count = 1 + int(elapsed_time // 30)
    if warning_count > max_warnings:
        warning_count = max_warnings

    # 일정 시간마다 새로운 위험구역 생성
    if current_ticks - last_warning_time >= warning_interval:
        warnings = []
        attempts = 0
        max_attempts = 500

        while len(warnings) < warning_count and attempts < max_attempts:
            warning_side = random.choice(["top", "bottom", "left", "right"])

            if warning_side == "top" or warning_side == "bottom":
                warning_pos = random.randint(0, WIDTH - warning_length)
            else:
                warning_pos = random.randint(0, HEIGHT - warning_length)

            if not is_too_close(warning_side, warning_pos, warnings, warning_length, min_gap):
                warnings.append(make_warning(warning_side, warning_pos, current_ticks))

            attempts += 1

        last_warning_time = current_ticks

    # 경고 후 1초 지나면 삼각형 발사
    for warning in warnings:
        if not warning["shot"]:
            if current_ticks - warning["created_time"] >= warning_delay:
                bullets.append(make_bullet(warning["side"], warning["pos"]))
                warning["shot"] = True

    # 삼각형 이동
    for bullet in bullets:
        bullet["x"] += bullet["vx"] * dt
        bullet["y"] += bullet["vy"] * dt

    # 화면 밖 나간 삼각형 제거
    new_bullets = []
    for bullet in bullets:
        if -50 <= bullet["x"] <= WIDTH + 50 and -50 <= bullet["y"] <= HEIGHT + 50:
            new_bullets.append(bullet)
    bullets = new_bullets

    screen.fill(WHITE)

    # 기본 벽
    pygame.draw.rect(screen, BLACK, (0, 0, WIDTH, HEIGHT), wall_thickness)

    # 빨간 위험구역 그리기
    for warning in warnings:
        warning_side = warning["side"]
        warning_pos = warning["pos"]

        if warning_side == "top":
            pygame.draw.rect(screen, RED, (warning_pos, 0, warning_length, wall_thickness))
        elif warning_side == "bottom":
            pygame.draw.rect(screen, RED, (warning_pos, HEIGHT - wall_thickness, warning_length, wall_thickness))
        elif warning_side == "left":
            pygame.draw.rect(screen, RED, (0, warning_pos, wall_thickness, warning_length))
        elif warning_side == "right":
            pygame.draw.rect(screen, RED, (WIDTH - wall_thickness, warning_pos, wall_thickness, warning_length))

    # 삼각형 장애물 그리기
    for bullet in bullets:
        points = get_triangle_points(bullet)
        pygame.draw.polygon(screen, RED, points)

    # 공 그리기
    pygame.draw.circle(screen, BLUE, (int(x), int(y)), radius)

    # FPS 표시
    fps = int(clock.get_fps())
    fps_text = font.render(f"FPS: {fps}", True, BLACK)
    screen.blit(fps_text, (10, 10))

    # 좌표 표시
    pos_text = font.render(f"X: {int(x)}  Y: {int(y)}", True, BLACK)
    pos_rect = pos_text.get_rect(topright=(WIDTH - 10, 10))
    screen.blit(pos_text, pos_rect)

    # 점수 표시
    score_text = font.render(f"Score: {int(score)}", True, BLACK)
    score_rect = score_text.get_rect(bottomright=(WIDTH - 10, HEIGHT - 10))
    screen.blit(score_text, score_rect)

    pygame.display.flip()

pygame.quit()
sys.exit()

# commit update
