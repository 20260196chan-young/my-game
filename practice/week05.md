import pygame
import random
import sys

pygame.init()


def get_korean_font(size):
    candidates = ["malgungothic", "applegothic", "nanumgothic", "notosanscjk"]
    for name in candidates:
        font = pygame.font.SysFont(name, size)
        if font.get_ascent() > 0:
            return font
    return pygame.font.SysFont(None, size)


WIDTH, HEIGHT = 800, 600
FPS = 60

WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
GRAY = (20, 20, 40)
BLUE = (50, 150, 255)
RED = (220, 50, 50)
YELLOW = (240, 220, 0)
GREEN = (50, 220, 80)
ORANGE = (240, 140, 0)
PURPLE = (170, 80, 255)
CYAN = (80, 220, 255)
LIGHT_CYAN = (180, 255, 255)

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Space Shooter")
clock = pygame.time.Clock()
font = get_korean_font(28)
font_big = get_korean_font(72)

# -----------------------------
# 스테이지 설정
# -----------------------------
STAGE_THRESHOLDS = [0, 100, 300, 700, 1500, 2500, 4000, 6500, 10000, 15000]

LEVELS = [
    {"enemy_speed": 2,  "spawn": 60, "label": "Stage 1",  "enemy_hp": 1},
    {"enemy_speed": 3,  "spawn": 55, "label": "Stage 2",  "enemy_hp": 2},
    {"enemy_speed": 4,  "spawn": 50, "label": "Stage 3",  "enemy_hp": 4},
    {"enemy_speed": 5,  "spawn": 45, "label": "Stage 4",  "enemy_hp": 4},
    {"enemy_speed": 6,  "spawn": 40, "label": "Stage 5",  "enemy_hp": 7},
    {"enemy_speed": 7,  "spawn": 36, "label": "Stage 6",  "enemy_hp": 7},
    {"enemy_speed": 8,  "spawn": 32, "label": "Stage 7",  "enemy_hp": 10},
    {"enemy_speed": 9,  "spawn": 28, "label": "Stage 8",  "enemy_hp": 12},
    {"enemy_speed": 10, "spawn": 24, "label": "Stage 9",  "enemy_hp": 15},
    {"enemy_speed": 11, "spawn": 20, "label": "Stage 10", "enemy_hp": 20},
]

PLAYER_W, PLAYER_H = 40, 40
ENEMY_W, ENEMY_H = 36, 36
BULLET_W, BULLET_H = 6, 14

BULLET_DAMAGE = 1

# -----------------------------
# 궁극기 설정
# -----------------------------
ULTIMATE_MAX = 100
ULTIMATE_GAIN_PER_KILL = 20
ULTIMATE_KEY = pygame.K_x

# -----------------------------
# 에너지파 설정
# -----------------------------
BEAM_DURATION = 90
BEAM_DAMAGE = 1
BEAM_TICK = 4
BEAM_WIDTH_OUTER = 90
BEAM_WIDTH_MIDDLE = 56
BEAM_WIDTH_CORE = 24


def get_level_index(score):
    for i in range(len(STAGE_THRESHOLDS) - 1, -1, -1):
        if score >= STAGE_THRESHOLDS[i]:
            return i
    return 0


def draw_player(surf, rect):
    cx = rect.centerx
    pygame.draw.polygon(surf, BLUE, [
        (cx, rect.top),
        (rect.left, rect.bottom),
        (cx, rect.bottom - 8),
        (rect.right, rect.bottom),
    ])
    pygame.draw.rect(surf, YELLOW, (cx - 4, rect.bottom - 10, 8, 10))


def draw_enemy(surf, enemy):
    rect = enemy["rect"]
    cx = rect.centerx

    if enemy["max_hp"] >= 10:
        body_color = ORANGE
    else:
        body_color = RED

    pygame.draw.polygon(surf, body_color, [
        (cx, rect.bottom),
        (rect.left, rect.top),
        (cx, rect.top + 8),
        (rect.right, rect.top),
    ])


def draw_enemy_hp(surf, enemy):
    rect = enemy["rect"]
    hp = enemy["hp"]
    max_hp = enemy["max_hp"]

    bar_width = rect.width
    bar_height = 5
    fill_width = int(bar_width * (hp / max_hp))

    pygame.draw.rect(surf, RED, (rect.left, rect.top - 8, bar_width, bar_height))
    pygame.draw.rect(surf, GREEN, (rect.left, rect.top - 8, fill_width, bar_height))


def spawn_enemy(level_cfg):
    x = random.randint(0, WIDTH - ENEMY_W)
    hp = level_cfg["enemy_hp"]
    return {
        "rect": pygame.Rect(x, -ENEMY_H, ENEMY_W, ENEMY_H),
        "hp": hp,
        "max_hp": hp
    }


def draw_stars(stars):
    for s in stars:
        pygame.draw.circle(screen, WHITE, (s[0], s[1]), s[2])


def draw_hud(score, lives, level_cfg, ultimate_gauge, ultimate_flash_timer):
    screen.blit(font.render(f"Score: {score}", True, WHITE), (10, 10))
    screen.blit(font.render(f"Lives: {'♥ ' * lives}", True, RED), (WIDTH - 180, 10))
    screen.blit(font.render(level_cfg["label"], True, YELLOW), (WIDTH // 2 - 60, 10))

    gauge_x = 20
    gauge_y = HEIGHT - 40
    gauge_w = 260
    gauge_h = 18

    pygame.draw.rect(screen, WHITE, (gauge_x - 2, gauge_y - 2, gauge_w + 4, gauge_h + 4), 2)
    pygame.draw.rect(screen, (40, 40, 60), (gauge_x, gauge_y, gauge_w, gauge_h))

    fill_w = int(gauge_w * (ultimate_gauge / ULTIMATE_MAX))
    pygame.draw.rect(screen, PURPLE, (gauge_x, gauge_y, fill_w, gauge_h))

    screen.blit(font.render("ULT", True, WHITE), (gauge_x + gauge_w + 15, gauge_y - 6))

    if ultimate_gauge >= ULTIMATE_MAX:
        if ultimate_flash_timer % 30 < 15:
            ready_text = font.render("READY! Press X", True, CYAN)
        else:
            ready_text = font.render("READY! Press X", True, WHITE)
        screen.blit(ready_text, (gauge_x + 330, gauge_y - 6))
    else:
        percent_text = font.render(f"{ultimate_gauge}%", True, WHITE)
        screen.blit(percent_text, (gauge_x + 330, gauge_y - 6))


def draw_beam(surf, player_rect, beam_timer):
    if beam_timer <= 0:
        return

    cx = player_rect.centerx
    top_y = 0
    bottom_y = player_rect.top

    pulse = (beam_timer % 6) - 3
    outer_w = BEAM_WIDTH_OUTER + abs(pulse) * 2
    middle_w = BEAM_WIDTH_MIDDLE + abs(pulse)
    core_w = BEAM_WIDTH_CORE + abs(pulse)

    outer_rect = pygame.Rect(cx - outer_w // 2, top_y, outer_w, bottom_y - top_y)
    middle_rect = pygame.Rect(cx - middle_w // 2, top_y, middle_w, bottom_y - top_y)
    core_rect = pygame.Rect(cx - core_w // 2, top_y, core_w, bottom_y - top_y)

    pygame.draw.rect(surf, CYAN, outer_rect)
    pygame.draw.rect(surf, LIGHT_CYAN, middle_rect)
    pygame.draw.rect(surf, WHITE, core_rect)

    pygame.draw.circle(surf, CYAN, (cx, player_rect.top), outer_w // 2)
    pygame.draw.circle(surf, LIGHT_CYAN, (cx, player_rect.top), middle_w // 2)
    pygame.draw.circle(surf, WHITE, (cx, player_rect.top), core_w // 2)


def get_beam_rect(player_rect):
    return pygame.Rect(
        player_rect.centerx - BEAM_WIDTH_OUTER // 2,
        0,
        BEAM_WIDTH_OUTER,
        player_rect.top
    )


def game_over_screen(score):
    screen.fill((10, 10, 30))
    screen.blit(font_big.render("GAME OVER", True, RED), (220, 220))
    screen.blit(font.render(f"Score: {score}", True, WHITE), (350, 310))
    screen.blit(font.render("R: Restart   Q: Quit", True, WHITE), (270, 360))
    pygame.display.flip()

    while True:
        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                pygame.quit()
                sys.exit()

            if e.type == pygame.KEYDOWN:
                if e.key == pygame.K_r:
                    return True
                if e.key == pygame.K_q:
                    pygame.quit()
                    sys.exit()


def main():
    player = pygame.Rect(WIDTH // 2 - PLAYER_W // 2, HEIGHT - 70, PLAYER_W, PLAYER_H)

    bullets = []
    enemies = []

    score = 0
    lives = 3

    shoot_cd = 0
    spawn_timer = 0

    level_idx = 0
    level_cfg = LEVELS[level_idx]

    invincible = 0

    ultimate_gauge = 0
    ultimate_flash_timer = 0

    beam_timer = 0
    beam_damage_timer = 0

    stars = [
        (random.randint(0, WIDTH), random.randint(0, HEIGHT), random.randint(1, 2))
        for _ in range(80)
    ]

    while True:
        clock.tick(FPS)
        ultimate_flash_timer += 1

        # -----------------------------
        # 이벤트 처리
        # -----------------------------
        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                pygame.quit()
                sys.exit()

            if e.type == pygame.KEYDOWN:
                if e.key == ULTIMATE_KEY and ultimate_gauge >= ULTIMATE_MAX and beam_timer <= 0:
                    ultimate_gauge = 0
                    beam_timer = BEAM_DURATION
                    beam_damage_timer = 0

        # -----------------------------
        # 입력 처리
        # -----------------------------
        keys = pygame.key.get_pressed()

        if keys[pygame.K_LEFT] and player.left > 0:
            player.x -= 6
        if keys[pygame.K_RIGHT] and player.right < WIDTH:
            player.x += 6
        if keys[pygame.K_UP] and player.top > 0:
            player.y -= 6
        if keys[pygame.K_DOWN] and player.bottom < HEIGHT:
            player.y += 6

        # -----------------------------
        # 일반 총알 발사
        # -----------------------------
        shoot_cd -= 1
        if keys[pygame.K_SPACE] and shoot_cd <= 0 and beam_timer <= 0:
            bullet = pygame.Rect(
                player.centerx - BULLET_W // 2,
                player.top,
                BULLET_W,
                BULLET_H
            )
            bullets.append(bullet)
            shoot_cd = 15

        # -----------------------------
        # 총알 이동
        # -----------------------------
        bullets = [b for b in bullets if b.bottom > 0]
        for b in bullets:
            b.y -= 10

        # -----------------------------
        # 적 생성
        # -----------------------------
        spawn_timer += 1
        if spawn_timer >= level_cfg["spawn"]:
            spawn_timer = 0
            enemies.append(spawn_enemy(level_cfg))

        # -----------------------------
        # 적 이동
        # -----------------------------
        alive_enemies = []
        for en in enemies:
            en["rect"].y += level_cfg["enemy_speed"]
            if en["rect"].top < HEIGHT:
                alive_enemies.append(en)
        enemies = alive_enemies

        # -----------------------------
        # 총알-적 충돌
        # -----------------------------
        hit_bullets = set()
        dead_enemies = set()

        for bi, b in enumerate(bullets):
            for ei, en in enumerate(enemies):
                if b.colliderect(en["rect"]):
                    hit_bullets.add(bi)
                    en["hp"] -= BULLET_DAMAGE

                    if en["hp"] <= 0:
                        dead_enemies.add(ei)
                        score += 10
                        ultimate_gauge += ULTIMATE_GAIN_PER_KILL
                        if ultimate_gauge > ULTIMATE_MAX:
                            ultimate_gauge = ULTIMATE_MAX
                    break

        bullets = [b for i, b in enumerate(bullets) if i not in hit_bullets]
        enemies = [en for i, en in enumerate(enemies) if i not in dead_enemies]

        # -----------------------------
        # 에너지파 데미지 처리
        # -----------------------------
        if beam_timer > 0:
            beam_timer -= 1
            beam_damage_timer += 1

            if beam_damage_timer >= BEAM_TICK:
                beam_damage_timer = 0
                beam_rect = get_beam_rect(player)

                dead_indices = set()
                for ei, en in enumerate(enemies):
                    if beam_rect.colliderect(en["rect"]):
                        en["hp"] -= BEAM_DAMAGE
                        if en["hp"] <= 0:
                            dead_indices.add(ei)
                            score += 10

                enemies = [en for i, en in enumerate(enemies) if i not in dead_indices]

        # -----------------------------
        # 스테이지 갱신
        # -----------------------------
        level_idx = get_level_index(score)
        level_cfg = LEVELS[level_idx]

        # -----------------------------
        # 플레이어 피격
        # -----------------------------
        if invincible > 0:
            invincible -= 1
        else:
            for en in enemies:
                if player.colliderect(en["rect"]):
                    lives -= 1
                    invincible = 90
                    enemies.clear()

                    if lives <= 0:
                        if game_over_screen(score):
                            main()
                        return
                    break

        # -----------------------------
        # 그리기
        # -----------------------------
        screen.fill(GRAY)
        draw_stars(stars)

        for b in bullets:
            pygame.draw.rect(screen, YELLOW, b)

        # 빔을 먼저 그림
        draw_beam(screen, player, beam_timer)

        # 적은 나중에 그려서 빔에 안 가려지게
        for en in enemies:
            draw_enemy(screen, en)
            draw_enemy_hp(screen, en)

        blink = (invincible // 10) % 2 == 0
        if blink:
            draw_player(screen, player)

        draw_hud(score, lives, level_cfg, ultimate_gauge, ultimate_flash_timer)
        pygame.display.flip()


main()