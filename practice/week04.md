Eimport pygame
import sys
import math
import base64
import io

pygame.init()

WIDTH = 800
HEIGHT = 600

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("충돌 방식 전환")

clock = pygame.time.Clock()
font = pygame.font.SysFont(None, 36)
hit_font = pygame.font.SysFont(None, 100)

# 색상
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)

SKYBLUE = (135, 206, 235)     # Circle
ORANGE = (255, 165, 0)        # AABB
LIGHTGREEN = (144, 238, 144)  # OBB

BLUE = (0, 0, 255)
RED = (255, 0, 0)
GREEN = (0, 200, 0)
AABB_LINE = (255, 0, 0)

# -------------------------------
# Base64 이미지 복붙
# -------------------------------
_ROCKET = (
    "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"
    "bmNvZGVyqAZ/7gAAAABJRU5ErkJggg=="
)

def load_rocket(size=None):
    raw = base64.b64decode(_ROCKET)
    buf = io.BytesIO(raw)
    surface = pygame.image.load(buf).convert_alpha()
    if size:
        surface = pygame.transform.scale(surface, size)
    return surface

moving_image = load_rocket((60, 160))
fixed_image = load_rocket((60, 160))

moving_x, moving_y = 200, 300
moving_angle = 0

fixed_x, fixed_y = WIDTH // 2, HEIGHT // 2
fixed_angle = 0

move_speed = 5
rotate_speed = 3
fixed_rotate_speed = 1

mode = 0  # 0:Circle, 1:AABB, 2:OBB

def get_obb(cx, cy, w, h, angle):
    hw, hh = w/2, h/2
    rad = math.radians(-angle)

    corners = [(-hw,-hh),(hw,-hh),(hw,hh),(-hw,hh)]
    pts = []

    for x,y in corners:
        rx = x*math.cos(rad) - y*math.sin(rad)
        ry = x*math.sin(rad) + y*math.cos(rad)
        pts.append((cx+rx, cy+ry))

    return pts

def normalize(x,y):
    l = math.sqrt(x*x+y*y)
    return (0,0) if l==0 else (x/l,y/l)

def dot(ax,ay,bx,by):
    return ax*bx + ay*by

def get_axes(p):
    axes=[]
    for i in range(len(p)):
        x1,y1=p[i]
        x2,y2=p[(i+1)%len(p)]
        ex,ey = x2-x1, y2-y1
        ax,ay = normalize(-ey,ex)
        axes.append((ax,ay))
    return axes

def project(p,axis):
    ax,ay = axis
    proj = dot(p[0][0],p[0][1],ax,ay)
    min_p=max_p=proj

    for x,y in p[1:]:
        val = dot(x,y,ax,ay)
        min_p=min(min_p,val)
        max_p=max(max_p,val)

    return min_p,max_p

def overlap(a_min,a_max,b_min,b_max):
    return a_max>=b_min and b_max>=a_min

def obb_collision(p1,p2):
    axes = get_axes(p1)+get_axes(p2)
    for axis in axes:
        min1,max1=project(p1,axis)
        min2,max2=project(p2,axis)
        if not overlap(min1,max1,min2,max2):
            return False
    return True

def circle_collision(x1,y1,r1,x2,y2,r2):
    dx=x1-x2
    dy=y1-y2
    return dx*dx+dy*dy <= (r1+r2)**2

def draw_text(text,x,y,color):
    img = font.render(text, True, color)
    screen.blit(img,(x,y))

def draw_hit():
    hit = hit_font.render("HIT", True, RED)
    # 🔥 여기 수정됨 (위쪽으로 올림)
    rect = hit.get_rect(center=(WIDTH//2, HEIGHT//4))
    screen.blit(hit, rect)

running=True
while running:
    for e in pygame.event.get():
        if e.type==pygame.QUIT:
            running=False
        if e.type==pygame.KEYDOWN:
            if e.key==pygame.K_f:
                mode=(mode+1)%3

    keys=pygame.key.get_pressed()

    if keys[pygame.K_LEFT]: moving_x-=move_speed
    if keys[pygame.K_RIGHT]: moving_x+=move_speed
    if keys[pygame.K_UP]: moving_y-=move_speed
    if keys[pygame.K_DOWN]: moving_y+=move_speed

    if keys[pygame.K_q]: moving_angle+=rotate_speed
    if keys[pygame.K_e]: moving_angle-=rotate_speed

    speed=fixed_rotate_speed
    if keys[pygame.K_z]: speed=3
    fixed_angle+=speed

    rot_m = pygame.transform.rotate(moving_image, moving_angle)
    rect_m = rot_m.get_rect(center=(moving_x,moving_y))

    rot_f = pygame.transform.rotate(fixed_image, fixed_angle)
    rect_f = rot_f.get_rect(center=(fixed_x,fixed_y))

    aabb_hit = rect_m.colliderect(rect_f)

    mw,mh = moving_image.get_size()
    fw,fh = fixed_image.get_size()

    moving_r = int(math.sqrt((mw/2)**2+(mh/2)**2))
    fixed_r = int(math.sqrt((fw/2)**2+(fh/2)**2))

    circle_hit = circle_collision(moving_x,moving_y,moving_r,fixed_x,fixed_y,fixed_r)

    obb_m = get_obb(moving_x,moving_y,mw,mh,moving_angle)
    obb_f = get_obb(fixed_x,fixed_y,fw,fh,fixed_angle)

    obb_hit = obb_collision(obb_m,obb_f)

    bg=WHITE
    if mode==0 and circle_hit: bg=SKYBLUE
    elif mode==1 and aabb_hit: bg=ORANGE
    elif mode==2 and obb_hit: bg=LIGHTGREEN

    screen.fill(bg)

    screen.blit(rot_f,rect_f)
    screen.blit(rot_m,rect_m)

    if mode==0:
        pygame.draw.circle(screen,BLUE,(fixed_x,fixed_y),fixed_r,2)
        pygame.draw.circle(screen,BLUE,(moving_x,moving_y),moving_r,2)
        draw_text("Mode: Circle",20,20,BLUE)
        draw_text(f"{'HIT' if circle_hit else 'MISS'}",20,60,BLUE)
        if circle_hit: draw_hit()

    elif mode==1:
        pygame.draw.rect(screen,AABB_LINE,rect_f,2)
        pygame.draw.rect(screen,AABB_LINE,rect_m,2)
        draw_text("Mode: AABB",20,20,AABB_LINE)
        draw_text(f"{'HIT' if aabb_hit else 'MISS'}",20,60,AABB_LINE)
        if aabb_hit: draw_hit()

    elif mode==2:
        pygame.draw.polygon(screen,GREEN,obb_f,2)
        pygame.draw.polygon(screen,GREEN,obb_m,2)
        draw_text("Mode: OBB",20,20,GREEN)
        draw_text(f"{'HIT' if obb_hit else 'MISS'}",20,60,GREEN)
        if obb_hit: draw_hit()

    draw_text("F: Change Mode",20,HEIGHT-40,BLACK)

    pygame.display.update()
    clock.tick(60)

pygame.quit()
sys.exit()