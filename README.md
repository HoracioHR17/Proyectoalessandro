
import pygame
from pygame.locals import *
from OpenGL.GL import *
from OpenGL.GLU import *
import math
import random

DISPLAY = (1200, 800)
G_ROWS = 5
G_COLS = 9
CELL_SIZE = 2.0

C_GRASS_1 = (0.2, 0.7, 0.2)
C_GRASS_2 = (0.15, 0.6, 0.15)
C_PEA_HEAD = (0.5, 0.95, 0.3)
C_ZOMBIE_SKIN = (0.7, 0.85, 0.65)
C_ZOMBIE_CLOTH = (0.3, 0.3, 0.6)
C_SUNFLOWER_CENTER = (0.9, 0.7, 0.2)
C_SUNFLOWER_PETAL = (1.0, 0.95, 0.3)
C_SUN = (1.0, 1.0, 0.5)
C_WALLNUT = (0.7, 0.5, 0.3)
C_SHADOW = (0.0, 0.0, 0.0, 0.25)
C_CURSOR_OK = (0.3, 1.0, 0.3, 0.6)
C_CURSOR_BAD = (1.0, 0.3, 0.3, 0.6)

text_textures = {}
quadric = None


def world_to_screen(x, y, z):
    try:
        modelview = glGetDoublev(GL_MODELVIEW_MATRIX)
        projection = glGetDoublev(GL_PROJECTION_MATRIX)
        viewport = glGetIntegerv(GL_VIEWPORT)
        winX, winY, winZ = gluProject(x, y, z, modelview, projection, viewport)
        return int(winX), int(DISPLAY[1] - winY)
    except:
        return -1000, -1000


def get_mouse_ray_ground(mx, my):
    try:
        viewport = glGetIntegerv(GL_VIEWPORT)
        modelview = glGetDoublev(GL_MODELVIEW_MATRIX)
        projection = glGetDoublev(GL_PROJECTION_MATRIX)
        winY = float(viewport[3] - my)
        
        p1 = gluUnProject(mx, winY, 0.0, modelview, projection, viewport)
        p2 = gluUnProject(mx, winY, 1.0, modelview, projection, viewport)
        
        dx, dy, dz = p2[0] - p1[0], p2[1] - p1[1], p2[2] - p1[2]
        
        if abs(dy) < 0.0001:
            return None, None
        
        t = -p1[1] / dy
        x = p1[0] + t * dx
        z = p1[2] + t * dz
        
        return x, z
    except:
        return None, None


def draw_shadow(scale=1.0):
    glDisable(GL_LIGHTING)
    glEnable(GL_BLEND)
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    glColor4fv(C_SHADOW)
    
    glPushMatrix()
    glTranslatef(0, 0.01, 0)
    glRotatef(-90, 1, 0, 0)
    glScalef(scale, scale, 1.0)
    gluDisk(quadric, 0, 0.5, 16, 1)
    glPopMatrix()
    
    glDisable(GL_BLEND)
    glEnable(GL_LIGHTING)

def draw_sphere(radius, r, g, b):
    glColor3f(r, g, b)
    gluSphere(quadric, radius, 20, 20)

def draw_cylinder(radius, height, r, g, b):
    glColor3f(r, g, b)
    gluCylinder(quadric, radius, radius, height, 20, 1)
    
    glPushMatrix()
    glRotatef(180, 1, 0, 0)
    gluDisk(quadric, 0, radius, 20, 1)
    glPopMatrix()
    
    glPushMatrix()
    glTranslatef(0, 0, height)
    gluDisk(quadric, 0, radius, 20, 1)
    glPopMatrix()

def draw_cursor_box(row, col, valid):
    x = (col - G_COLS/2) * CELL_SIZE
    z = (row - G_ROWS/2) * CELL_SIZE

    glDisable(GL_LIGHTING)
    glEnable(GL_BLEND)
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)

    glColor4fv(C_CURSOR_OK if valid else C_CURSOR_BAD)
    glPushMatrix()
    glTranslatef(x, 0.02, z)

    glBegin(GL_QUADS)
    glVertex3f(0, 0, 0)
    glVertex3f(CELL_SIZE, 0, 0)
    glVertex3f(CELL_SIZE, 0, CELL_SIZE)
    glVertex3f(0, 0, CELL_SIZE)
    glEnd()

    glLineWidth(4)
    glColor3f(1, 1, 1)
    glBegin(GL_LINE_LOOP)
    glVertex3f(0, 0, 0)
    glVertex3f(CELL_SIZE, 0, 0)
    glVertex3f(CELL_SIZE, 0, CELL_SIZE)
    glVertex3f(0, 0, CELL_SIZE)
    glEnd()

    glPopMatrix()
    glDisable(GL_BLEND)
    glEnable(GL_LIGHTING)


def create_text_texture(font, text, color=(255, 255, 255)):
    key = f"{text}_{color}"
    if key in text_textures:
        return text_textures[key]

    surface = font.render(text, True, color)
    w, h = surface.get_width(), surface.get_height()
    
    text_data = pygame.image.tostring(surface, "RGBA", 0)

    tid = glGenTextures(1)
    glBindTexture(GL_TEXTURE_2D, tid)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, w, h, 0, GL_RGBA, GL_UNSIGNED_BYTE, text_data)

    text_textures[key] = (tid, w, h)
    return tid, w, h


def draw_text_2d(x, y, tid, w, h):
    glEnable(GL_TEXTURE_2D)
    glBindTexture(GL_TEXTURE_2D, tid)
    glEnable(GL_BLEND)
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    glColor4f(1, 1, 1, 1)

    glBegin(GL_QUADS)
    glTexCoord2f(0, 0); glVertex2f(x, y)
    glTexCoord2f(1, 0); glVertex2f(x + w, y)
    glTexCoord2f(1, 1); glVertex2f(x + w, y + h)
    glTexCoord2f(0, 1); glVertex2f(x, y + h)
    glEnd()

    glDisable(GL_BLEND)
    glDisable(GL_TEXTURE_2D)


def draw_rect_2d(x, y, w, h, color, outline=False):
    glDisable(GL_LIGHTING)
    glColor3fv(color)

    if outline:
        glLineWidth(3)
        glBegin(GL_LINE_LOOP)
    else:
        glBegin(GL_QUADS)

    glVertex2f(x, y)
    glVertex2f(x + w, y)
    glVertex2f(x + w, y + h)
    glVertex2f(x, y + h)
    glEnd()

    glEnable(GL_LIGHTING)

class Plant:
    def __init__(self, row, col):
        self.row, self.col = row, col
        self.x = (col - G_COLS/2) * CELL_SIZE + (CELL_SIZE/2)
        self.z = (row - G_ROWS/2) * CELL_SIZE + (CELL_SIZE/2)
        self.hp = 5
        self.scale = 0.1

    def update(self):
        if self.scale < 1.0:
            self.scale = min(1.0, self.scale + 0.05)

class Sunflower(Plant):
    def __init__(self, row, col):
        super().__init__(row, col)
        self.timer = 0
        self.rot = 0
        self.hp = 4
    
    def update(self, suns):
        super().update()
        self.timer += 1
        self.rot += 2
        
        if self.timer > 480:
            self.timer = 0
            suns.append(Sun(self.x, 1.5, self.z, falls=False))

    def draw(self):
        glPushMatrix()
        glTranslatef(self.x, 0, self.z)
        draw_shadow(0.8)
        glScalef(self.scale, self.scale, self.scale)
        
        glPushMatrix()
        glRotatef(-90, 1, 0, 0)
        draw_cylinder(0.1, 0.9, 0.1, 0.7, 0.1)
        glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0, 1.0, 0)
        glRotatef(180, 0, 1, 0)
        glRotatef(self.rot, 0, 0, 1)
        
        for i in range(12):
            glPushMatrix()
            glRotatef(i * 30, 0, 0, 1)
            glTranslatef(0.45, 0, 0)
            glScalef(0.35, 0.18, 0.08)
            draw_sphere(1, *C_SUNFLOWER_PETAL)
            glPopMatrix()
        
        draw_sphere(0.35, *C_SUNFLOWER_CENTER)
        glPopMatrix()
        glPopMatrix()

class Peashooter(Plant):
    def __init__(self, row, col):
        super().__init__(row, col)
        self.timer = 0
        self.hp = 6
        self.recoil = 0

    def update(self, peas, zombies):
        super().update()
        self.timer += 1
        
        if self.recoil > 0:
            self.recoil = max(0, self.recoil - 0.05)
        
        has_target = any(z.row == self.row and z.x > self.x for z in zombies)
        
        if has_target and self.timer > 80:
            self.timer = 0
            self.recoil = 0.4
            peas.append(Pea(self.x + 0.6, 1.0, self.z))

    def draw(self):
        glPushMatrix()
        glTranslatef(self.x, 0, self.z)
        draw_shadow(0.7)
        glScalef(self.scale, self.scale, self.scale)
        
        glPushMatrix()
        glRotatef(-90, 1, 0, 0)
        draw_cylinder(0.1, 0.85, 0.1, 0.75, 0.1)
        glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0, 1.0, -self.recoil)
        draw_sphere(0.38, *C_PEA_HEAD)
        
        glPushMatrix()
        glTranslatef(0.35, 0, 0)
        glRotatef(90, 0, 1, 0)
        draw_cylinder(0.14, 0.35, *C_PEA_HEAD)
        glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0.25, 0.1, 0.15)
        draw_sphere(0.08, 0.1, 0.1, 0.1)
        glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0.25, 0.1, -0.15)
        draw_sphere(0.08, 0.1, 0.1, 0.1)
        glPopMatrix()
        glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0, 0.1, 0)
        glScalef(0.8, 0.15, 0.8)
        draw_sphere(0.5, 0.1, 0.6, 0.1)
        glPopMatrix()
        glPopMatrix()

class Wallnut(Plant):
    def __init__(self, row, col):
        super().__init__(row, col)
        self.hp = 40
        self.max_hp = 40
        self.shake = 0

    def update(self):
        super().update()
        if self.shake > 0:
            self.shake -= 1

    def draw(self):
        offset = math.sin(self.shake * 0.5) * 0.15 if self.shake > 0 else 0
        
        glPushMatrix()
        glTranslatef(self.x + offset, 0, self.z)
        draw_shadow(1.0)
        glScalef(self.scale, self.scale, self.scale)
        
        glPushMatrix()
        glTranslatef(0, 0.65, 0)
        
        damage_ratio = 1.0 - (self.hp / self.max_hp)
        r = C_WALLNUT[0] + damage_ratio * 0.35
        g = C_WALLNUT[1] - damage_ratio * 0.2
        b = C_WALLNUT[2]
        
        draw_sphere(0.55, r, g, b)
        
        glPushMatrix()
        glRotatef(45, 0, 1, 0)
        glScalef(1.05, 0.08, 1.05)
        draw_sphere(0.55, r - 0.2, g - 0.2, b - 0.1)
        glPopMatrix()
        
        glPushMatrix()
        glRotatef(-45, 0, 1, 0)
        glScalef(1.05, 0.08, 1.05)
        draw_sphere(0.55, r - 0.2, g - 0.2, b - 0.1)
        glPopMatrix()
        glPopMatrix()
        glPopMatrix()

class Pea:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
        self.active = True
        self.rot = 0

    def update(self):
        self.x += 0.18
        self.rot += 15
        if self.x > 18:
            self.active = False

    def draw(self):
        glPushMatrix()
        glTranslatef(self.x, self.y, self.z)
        glRotatef(self.rot, 1, 0, 0)
        draw_sphere(0.12, 0.2, 1.0, 0.2)
        glPopMatrix()

class Sun:
    def __init__(self, x, y, z, falls=True):
        self.x, self.y, self.z = x, y, z
        self.target_y = 0.6
        self.active = True
        self.falls = falls
        self.lifetime = 600
        self.rot = 0
        self.pulse = 0

    def update(self):
        self.rot += 3
        self.pulse += 0.15
        
        if self.falls and self.y > self.target_y:
            self.y -= 0.04
        
        self.lifetime -= 1
        if self.lifetime <= 0:
            self.active = False

    def draw(self):
        scale = 1.0 + math.sin(self.pulse) * 0.12
        
        glPushMatrix()
        glTranslatef(self.x, self.y, self.z)
        glRotatef(self.rot, 0, 0, 1)
        glScalef(scale, scale, scale)
        
        for i in range(8):
            glPushMatrix()
            glRotatef(i * 45, 0, 0, 1)
            glTranslatef(0.35, 0, 0)
            glScalef(0.25, 0.08, 0.08)
            draw_sphere(1.0, 1.0, 1.0, 0.7)
            glPopMatrix()
        
        draw_sphere(0.28, *C_SUN)
        glPopMatrix()

class Zombie:
    def __init__(self, row, variant=0):
        self.row = row
        self.col = G_COLS
        self.x = (self.col - G_COLS/2) * CELL_SIZE + CELL_SIZE
        self.z = (row - G_ROWS/2) * CELL_SIZE + (CELL_SIZE/2)
        self.hp = 12 if variant == 0 else 30
        self.speed = 0.018 if variant == 0 else 0.012
        self.variant = variant
        self.eating = False
        self.walk_anim = 0
        self.flash = 0

    def update(self, plants):
        if self.flash > 0:
            self.flash -= 1
        
        self.eating = False
        
        for p in plants[:]:
            if p.row == self.row and abs(p.x - self.x) < 0.7:
                self.eating = True
                p.hp -= 0.08
                
                if isinstance(p, Wallnut):
                    p.shake = 10
                
                if p.hp <= 0:
                    plants.remove(p)
                break
        
        if not self.eating:
            self.x -= self.speed
            self.walk_anim += 0.25

    def draw(self):
        glPushMatrix()
        glTranslatef(self.x, 0, self.z)
        glRotatef(180, 0, 1, 0)
        draw_shadow(1.1)
        
        skin = (1.0, 0.3, 0.3) if self.flash > 0 else C_ZOMBIE_SKIN
        cloth = (1.0, 0.3, 0.3) if self.flash > 0 else C_ZOMBIE_CLOTH
        
        swing = math.sin(self.walk_anim) * 25 if not self.eating else 0
        
        for i, z_offset in enumerate([0.18, -0.18]):
            glPushMatrix()
            glTranslatef(0, 0.7, z_offset)
            glRotatef(swing if i == 0 else -swing, 1, 0, 0)
            glTranslatef(0, -0.35, 0)
            glScalef(0.12, 0.7, 0.12)
            draw_sphere(1, *cloth)
            glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0, 1.3, 0)
        glScalef(0.28, 0.55, 0.35)
        draw_sphere(1, *cloth)
        glPopMatrix()
        
        glPushMatrix()
        glTranslatef(0, 2.0, 0)
        draw_sphere(0.32, *skin)
        
        for z_offset in [0.12, -0.12]:
            glPushMatrix()
            glTranslatef(0.22, 0.08, z_offset)
            draw_sphere(0.06, 1, 1, 1)
            glTranslatef(0.02, 0, 0)
            draw_sphere(0.03, 0, 0, 0)
            glPopMatrix()
        
        if self.variant == 1:
            glPushMatrix()
            glTranslatef(0, 0.32, 0)
            glRotatef(-90, 1, 0, 0)
            draw_cylinder(0.18, 0.55, 1.0, 0.6, 0.1)
            glPopMatrix()
        glPopMatrix()
        
        arm_angle = math.sin(self.walk_anim * 2) * 20 if self.eating else -80
        
        for z_offset in [0.15, -0.15]:
            glPushMatrix()
            glTranslatef(0.25, 1.5, z_offset)
            glRotatef(arm_angle, 0, 0, 1)
            glScalef(0.08, 0.5, 0.08)
            draw_sphere(1, *skin)
            glPopMatrix()
        
        glPopMatrix()


def main():
    global quadric
    
    pygame.init()
    screen = pygame.display.set_mode(DISPLAY, DOUBLEBUF | OPENGL)
    pygame.display.set_caption("Plants vs Zombies 3D")
    
    quadric = gluNewQuadric()
    gluQuadricNormals(quadric, GLU_SMOOTH)
    
    font = pygame.font.SysFont('Arial', 26, bold=True)
    font_go = pygame.font.SysFont('Arial', 72, bold=True)
    
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    gluPerspective(45, DISPLAY[0]/DISPLAY[1], 0.1, 100.0)
    glMatrixMode(GL_MODELVIEW)
    
    glEnable(GL_DEPTH_TEST)
    glEnable(GL_LIGHTING)
    glEnable(GL_LIGHT0)
    glEnable(GL_COLOR_MATERIAL)
    glColorMaterial(GL_FRONT_AND_BACK, GL_AMBIENT_AND_DIFFUSE)
    
    glLightfv(GL_LIGHT0, GL_POSITION, (5, 20, 10, 0))
    glLightfv(GL_LIGHT0, GL_AMBIENT, (0.6, 0.6, 0.6, 1))
    glLightfv(GL_LIGHT0, GL_DIFFUSE, (1.0, 1.0, 1.0, 1))
    
    clock = pygame.time.Clock()
    
    sun_count = 150
    selected_idx = None
    
    inventory = [
        ("Girasol", 50, (1, 0.9, 0)),
        ("Lanzaguisantes", 100, (0.3, 0.9, 0.3)),
        ("Almendra", 50, (0.85, 0.6, 0.35))
    ]
    
    plants = []
    zombies = []
    peas = []
    suns = []
    
    timer_spawn = 0
    timer_sun = 0
    wave = 1.0
    game_over = False
    
    running = True
    while running:
        clock.tick(60)
        mx, my = pygame.mouse.get_pos()
        
        for event in pygame.event.get():
            if event.type == QUIT:
                running = False
            
            if event.type == KEYDOWN:
                if event.key == K_ESCAPE:
                    if selected_idx is not None:
                        selected_idx = None
                    else:
                        running = False
                
                if event.key == K_r and game_over:
                    plants, zombies, peas, suns = [], [], [], []
                    sun_count = 150
                    wave = 1.0
                    timer_spawn = 0
                    timer_sun = 0
                    game_over = False
                
                if not game_over:
                    if event.key == K_1: selected_idx = 0
                    if event.key == K_2: selected_idx = 1
                    if event.key == K_3: selected_idx = 2
            
            if event.type == MOUSEBUTTONDOWN and event.button == 1 and not game_over:
                if my < 80:
                    for i in range(3):
                        bx = 200 + i * 160
                        if bx < mx < bx + 140:
                            selected_idx = i
                else:
                    sun_clicked = False
                    for s in suns[:]:
                        sx, sy = world_to_screen(s.x, s.y, s.z)
                        dist = math.hypot(mx - sx, my - sy)
                        if dist < 45:
                            suns.remove(s)
                            sun_count += 45
                            sun_clicked = True
                            break
                    
                    if not sun_clicked and selected_idx is not None:
                        wx, wz = get_mouse_ray_ground(mx, my)
                        if wx is not None and wz is not None:
                            col = int((wx + (G_COLS * CELL_SIZE) / 2) / CELL_SIZE)
                            row = int((wz + (G_ROWS * CELL_SIZE) / 2) / CELL_SIZE)
                            
                            if 0 <= row < G_ROWS and 0 <= col < G_COLS:
                                occupied = any(p.row == row and p.col == col for p in plants)
                                cost = inventory[selected_idx][1]
                                
                                if not occupied and sun_count >= cost:
                                    sun_count -= cost
                                    
                                    if selected_idx == 0:
                                        plants.append(Sunflower(row, col))
                                    elif selected_idx == 1:
                                        plants.append(Peashooter(row, col))
                                    elif selected_idx == 2:
                                        plants.append(Wallnut(row, col))
                                    
                                    selected_idx = None

        if not game_over:
            timer_spawn += 1
            timer_sun += 1
            
            spawn_interval = max(80, 420 - int(wave) * 25)
            if timer_spawn > spawn_interval:
                variant = 1 if wave > 3 and random.random() < 0.35 else 0
                zombies.append(Zombie(random.randint(0, G_ROWS - 1), variant))
                timer_spawn = 0
                wave += 0.15
            
            if timer_sun > 380:
                x = random.uniform(-7, 7)
                z = random.uniform(-3.5, 3.5)
                suns.append(Sun(x, 12, z, falls=True))
                timer_sun = 0
            
            for p in plants[:]:
                if isinstance(p, Peashooter):
                    p.update(peas, zombies)
                elif isinstance(p, Sunflower):
                    p.update(suns)
                else:
                    p.update()
            
            for z in zombies[:]:
                z.update(plants)
            
            for p in peas[:]:
                p.update()
            
            for s in suns[:]:
                s.update()
            
            for pea in peas[:]:
                if not pea.active:
                    continue
                
                for zombie in zombies[:]:
                    if abs(pea.z - zombie.z) < 0.4 and abs(pea.x - zombie.x) < 0.5:
                        zombie.hp -= 3
                        zombie.flash = 8
                        pea.active = False
                        
                        if zombie.hp <= 0:
                            if zombie in zombies:
                                zombies.remove(zombie)
                            sun_count += 15
                        break
            
            peas = [p for p in peas if p.active]
            suns = [s for s in suns if s.active]
            
            if any(z.x < -9 for z in zombies):
                game_over = True

        glClearColor(0.55, 0.75, 0.95, 1.0)
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
        glLoadIdentity()
        
        gluLookAt(0, 12, 14, 0, 0, 0, 0, 1, 0)
        
        glBegin(GL_QUADS)
        glNormal3f(0, 1, 0)
        for r in range(G_ROWS):
            for c in range(G_COLS):
                color = C_GRASS_1 if (r + c) % 2 == 0 else C_GRASS_2
                glColor3fv(color)
                
                x = (c - G_COLS/2) * CELL_SIZE
                z = (r - G_ROWS/2) * CELL_SIZE
                
                glVertex3f(x, 0, z)
                glVertex3f(x + CELL_SIZE, 0, z)
                glVertex3f(x + CELL_SIZE, 0, z + CELL_SIZE)
                glVertex3f(x, 0, z + CELL_SIZE)
        glEnd()
        
        if not game_over and selected_idx is not None:
            wx, wz = get_mouse_ray_ground(mx, my)
            if wx is not None and wz is not None:
                col = int((wx + (G_COLS * CELL_SIZE) / 2) / CELL_SIZE)
                row = int((wz + (G_ROWS * CELL_SIZE) / 2) / CELL_SIZE)
                
                if 0 <= row < G_ROWS and 0 <= col < G_COLS:
                    occupied = any(p.row == row and p.col == col for p in plants)
                    cost = inventory[selected_idx][1]
                    valid = not occupied and sun_count >= cost
                    draw_cursor_box(row, col, valid)
        
        for entity in plants + zombies + peas + suns:
            entity.draw()
        
        glMatrixMode(GL_PROJECTION)
        glPushMatrix()
        glLoadIdentity()
        glOrtho(0, DISPLAY[0], DISPLAY[1], 0, -1, 1)
        
        glMatrixMode(GL_MODELVIEW)
        glPushMatrix()
        glLoadIdentity()
        
        glDisable(GL_DEPTH_TEST)
        glDisable(GL_LIGHTING)
        
        draw_rect_2d(0, 0, DISPLAY[0], 85, (0.2, 0.15, 0.1))
        
        sun_tex = create_text_texture(font, f"SOLES: {sun_count}", (255, 240, 100))
        draw_text_2d(25, 32, *sun_tex)
        
        for i, (name, cost, col) in enumerate(inventory):
            bx = 200 + i * 160
            
            button_col = col if sun_count >= cost else (0.25, 0.25, 0.25)
            draw_rect_2d(bx, 12, 140, 62, button_col)
            
            if selected_idx == i:
                draw_rect_2d(bx, 12, 140, 62, (1, 1, 1), outline=True)
            
            name_tex = create_text_texture(font, name[:8], (255, 255, 255))
            cost_tex = create_text_texture(font, f"{cost}", (255, 255, 100))
            draw_text_2d(bx + 12, 18, *name_tex)
            draw_text_2d(bx + 12, 48, *cost_tex)
        
        if game_over:
            go_tex = create_text_texture(font_go, "GAME OVER", (255, 50, 50))
            restart_tex = create_text_texture(font, "Presiona 'R' para Reiniciar", (255, 255, 255))
            
            draw_text_2d(DISPLAY[0]//2 - go_tex[1]//2, DISPLAY[1]//2 - 60, *go_tex)
            draw_text_2d(DISPLAY[0]//2 - restart_tex[1]//2, DISPLAY[1]//2 + 40, *restart_tex)
        
        glEnable(GL_DEPTH_TEST)
        glEnable(GL_LIGHTING)
        
        glMatrixMode(GL_PROJECTION)
        glPopMatrix()
        glMatrixMode(GL_MODELVIEW)
        glPopMatrix()
        
        pygame.display.flip()
    
    pygame.quit()

if __name__ == "__main__":
    main()
