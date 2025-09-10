# 423-project
from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLU import *
import random
import math
import time

# Window settings
WINDOW_WIDTH, WINDOW_HEIGHT = 1600,900
GRID_LENGTH = 600
fovY = 120

# Camera settings
camera_pos = (0, 500, 500)
first_person_view = False

# Player variables
gun_angle = 0
player_pos = [0, 200, 0]  # Start closer to healing center, Z=0 for alignment
player_max_health = 100
player_health = 100
weapon_level = 1
speed_boost_active = False
speed_boost_end_frame = 0
shield_active = False
shield_end_frame = 0
cheat_mode = False

# Game stats
score = 0
total_points = 0
missed_bullets = 0
lives = 10
game_over = False
level = 1
last_level_up_score = 0
score_multiplier = 1
frame_counter = 0
last_enemy_spawn_frame = 0
last_boss_spawn_frame = 0
last_auto_fire_frame = 0
last_manual_fire_frame = 0 # MODIFIED: Added for manual fire rate control

# Game objects - ALL use Z=0 for perfect alignment
enemies = []  # [x, y, z, size, type]
enemy_spaceships = []  # [x, y, z, health, last_shoot_frame]
bullets = []  # [x, y, z, angle, speed]
enemy_bullets = []  # [x, y, z, angle, speed]
power_ups = []  # [x, y, z, type, rotation, creation_frame]
resources = []  # [x, y, z, rotation, creation_frame] # MODIFIED: Added creation_frame
explosions = []  # [x, y, z, size, frames_left]
boss_enemies = []  # [x, y, z, health, last_shoot_frame, movement_phase]
# Animation variables
earth_angle = 0
moon_angle = 0

# Power-up types
HEALTH_POWERUP = 0
WEAPON_POWERUP = 1
SPEED_POWERUP = 2
SHIELD_POWERUP = 3

# Power-up management constants
MAX_POWERUPS = 5
POWERUP_LIFETIME_FRAMES = 900  # 15 seconds at 60 FPS

# MODIFIED: Resource management constants
MAX_RESOURCES = 5
MIN_RESOURCES = 2
RESOURCE_LIFETIME_FRAMES = 300 # 5 seconds at 60 FPS

# CRITICAL FIX: All gameplay objects at Z=0 for perfect alignment


#Promit's part
##################################################################################
def draw_text(x, y, text, font=GLUT_BITMAP_HELVETICA_18):
    glColor3f(1, 1, 1) #white
    glMatrixMode(GL_PROJECTION)
    glPushMatrix()
    glLoadIdentity()
    gluOrtho2D(0, WINDOW_WIDTH, 0, WINDOW_HEIGHT)
    glMatrixMode(GL_MODELVIEW)
    glPushMatrix()
    glLoadIdentity()
    glRasterPos2f(x, y)
    for ch in text:
        glutBitmapCharacter(font, ord(ch))
    glPopMatrix()
    glMatrixMode(GL_PROJECTION)
    glPopMatrix()
    glMatrixMode(GL_MODELVIEW)
def draw_health_bar():
    bar_width = 250
    bar_height = 25
    bar_x = 15
    bar_y = WINDOW_HEIGHT - 200
    
    glMatrixMode(GL_PROJECTION)
    glPushMatrix()
    glLoadIdentity()
    gluOrtho2D(0, WINDOW_WIDTH, 0, WINDOW_HEIGHT)
    glMatrixMode(GL_MODELVIEW)
    glPushMatrix()
    glLoadIdentity()
    
    # Health bar color
    if player_health  > 60:
        glColor3f(0, 1, 0)
    elif player_health  > 30:
        glColor3f(1, 1, 0)
    else:
        glColor3f(1, 0, 0)
    # Health bar 
    glBegin(GL_QUADS)
    glVertex2f(bar_x, bar_y)
    glVertex2f(bar_x + bar_width * player_health/100, bar_y)
    glVertex2f(bar_x + bar_width * player_health/100, bar_y + bar_height)
    glVertex2f(bar_x, bar_y + bar_height)
    glEnd()
    
    glPopMatrix()
    glMatrixMode(GL_PROJECTION)
    glPopMatrix()
    glMatrixMode(GL_MODELVIEW)
def draw_mini_map():
    map_x = WINDOW_WIDTH - 200 # bottom-right corner
    map_y = 10
    
    # Disable depth test so map draws on top
    glDisable(GL_DEPTH_TEST)
    
    glMatrixMode(GL_PROJECTION)
    glPushMatrix()
    glLoadIdentity()
    gluOrtho2D(0, WINDOW_WIDTH, 0, WINDOW_HEIGHT)
    glMatrixMode(GL_MODELVIEW)
    glPushMatrix()
    glLoadIdentity()
    
    # Map background 
    glColor3f(0, 0, 0)
    glBegin(GL_QUADS)
    glVertex2f(map_x, map_y)
    glVertex2f(map_x + 150, map_y)
    glVertex2f(map_x + 150, map_y + 150)
    glVertex2f(map_x, map_y + 150)
    glEnd()
    
    # Scale factor for map
    scale = 150 / (GRID_LENGTH * 2)  #scale = minimap square side / grid square side
    center_x = map_x + 75   #map center (x,y) =150/2
    center_y = map_y + 75
    
    # Draw healing center in map
    glColor3f(0, 1, 0)
    glPointSize(5)
    glBegin(GL_POINTS)
    glVertex2f(center_x, center_y)
    glEnd()
    
    # Draw player in map 
    glColor3f(0, 1, 1) #cyan
    glPointSize(5)
    glBegin(GL_POINTS)
    #scaling player position in grid to map
    glVertex2f(center_x - player_pos[0] * scale, center_y - player_pos[1] * scale) 
    glEnd()
    
    # Draw enemies 
    glColor3f(1, 0, 0) # red
    glPointSize(4)
    glBegin(GL_POINTS)
    for i in enemies :
        glVertex2f(center_x - i[0] * scale, center_y - i[1] * scale)
    
    for j in enemy_spaceships:
        glVertex2f(center_x - j[0] * scale, center_y - j[1] * scale)
    
    for k in boss_enemies:
        glVertex2f(center_x - k[0] * scale, center_y - k[1] * scale)
    glEnd()
    
    glPopMatrix()
    glMatrixMode(GL_PROJECTION)
    glPopMatrix()
    glMatrixMode(GL_MODELVIEW)
    
    # Re-enable depth test for 3D world
    glEnable(GL_DEPTH_TEST)
def draw_power_up(x, y, z, power_type, rotation):
    glPushMatrix()
    glTranslatef(x, y, z)
    glRotatef(rotation, 1, 1, 0)
    
    if power_type == HEALTH_POWERUP:
        # Red health cube 
        glColor3f(1, 0, 0) 
        glutSolidCube(25)   
    elif power_type == WEAPON_POWERUP:
        # Yellow weapon cube 
        glColor3f(1, 1, 0)
        glutSolidCube(25)
    elif power_type == SPEED_POWERUP:
        # Blue speed cube 
        glColor3f(0, 0.5, 1)
        glutSolidCube(25)
    elif power_type == SHIELD_POWERUP:
        # Purple shield cube
        glColor3f(1, 0, 1)
        glutSolidCube(25)
        glColor3f(0.7, 0, 1)
        gluSphere(gluNewQuadric(), 18, 12, 12) # (radius, longitudinal slices, latitude slices)

    glPopMatrix()
def update_power_ups():
    global power_ups, player_health, weapon_level, speed_boost_active, speed_boost_end_frame, shield_active, shield_end_frame, frame_counter
    
    power_ups[:] = [p for p in power_ups if frame_counter - p[5] <= POWERUP_LIFETIME_FRAMES]

    for p in power_ups[:]:
        p[4] += 4
        
        dist = math.sqrt((p[0] - player_pos[0])**2 + (p[1] - player_pos[1])**2)
        if dist < 50:
            power_ups.remove(p)
            
            if p[3] == HEALTH_POWERUP:
                player_health = min(player_max_health, player_health + 35)
            elif p[3] == WEAPON_POWERUP:
                weapon_level = min(5, weapon_level + 1)
            elif p[3] == SPEED_POWERUP:
                speed_boost_active = True
                speed_boost_end_frame = frame_counter + 600
            elif p[3] == SHIELD_POWERUP:
                shield_active = True
                shield_end_frame = frame_counter + 720
def setupCamera():
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    gluPerspective(fovY, WINDOW_WIDTH / WINDOW_HEIGHT, 0.1, 1000)
    glMatrixMode(GL_MODELVIEW)
    glLoadIdentity()
    
    if first_person_view:
        # First person view
        look_x = player_pos[0] + 100 * math.cos(math.radians(gun_angle))
        look_y = player_pos[1] + 100 * math.sin(math.radians(gun_angle))
        look_z = player_pos[2]
        
        pos_x = player_pos[0] - 10 * math.cos(math.radians(gun_angle))
        pos_y = player_pos[1] - 10 * math.sin(math.radians(gun_angle))
        pos_z = player_pos[2] + 70
        
        gluLookAt(pos_x, pos_y, pos_z, look_x, look_y, look_z, 0, 0, 1)
    else:
        # Third person view
        x, y, z = camera_pos
        gluLookAt(x, y, z, 0, 0, 0, 0, 0, 1) #camera at (x,y,z), looking at origin
#look in keyboardListener for speedbooster
###################################################################################

#Nashid's part
###############################################################################
def draw_healing_center():
    #Large, rotating healing center at origin - much more visible
    glPushMatrix()
    glTranslatef(0, 0, 0) #object drawn at origin
    
    # Outer healing ring - bright green, larger
    glColor3f(0, 1, 0) #green
    glPushMatrix()
    glRotatef(90, 1, 0, 0)  
    glRotatef(earth_angle * 2, 0, 0, 1)
    gluCylinder(gluNewQuadric(), 80, 80, 8, 24, 1)  # Larger and more segments
    glPopMatrix()
    
    # Middle healing ring
    glColor3f(0, 0.9, 0) #darker green
    glPushMatrix()
    glRotatef(90, 1, 0, 0)
    glRotatef(-earth_angle * 3, 0, 0, 1)
    gluCylinder(gluNewQuadric(), 60, 60, 8, 20, 1)
    glPopMatrix()
    # Inner healing ring
    glColor3f(0, 0.8, 0)
    glPushMatrix()
    glRotatef(90, 1, 0, 0)
    glRotatef(earth_angle * 4, 0, 0, 1)
    gluCylinder(gluNewQuadric(), 40, 40, 8, 16, 1)
    glPopMatrix()
    # Central healing sphere gluNewQuadric(),- larger and more visible
    glColor3f(0, 1, 0.5)
    gluSphere(gluNewQuadric(),10, 20, 20)
    
    # Enhanced healing particles effect - larger radius
    dist_to_center = math.sqrt(player_pos[0]**2 + player_pos[1]**2)
    if dist_to_center < 150:  # Increased healing radius
        glColor3f(0, 1, 0.7)
        for i in range(12):  # More particles
            angle = i * 30 + earth_angle * 5
            x = 70 * math.cos(math.radians(angle))
            y = 70 * math.sin(math.radians(angle))
            glPushMatrix()
            glTranslatef(x, y, 15)
            gluSphere(gluNewQuadric(),4, 10, 10)
            glPopMatrix()
    
    glPopMatrix()
def draw_enemy_spaceship(x, y, z, health):
    glPushMatrix()
    glTranslatef(x, y, z)
    
    # Health-based color gradation
    health_ratio = health / 30.0
    glColor3f(1, 0.2 + 0.8 * health_ratio, 0.2 + 0.8 * health_ratio)
    
    # Main body - slightly larger
    glutSolidCube(22)
    
    # Wings - more prominent
    glColor3f(0.8, 0, 0)
    glPushMatrix()
    glTranslatef(18, 0, 0)
    glScalef(0.6, 2.5, 0.4)
    glutSolidCube(12)
    glPopMatrix()
    glPushMatrix()
    glTranslatef(-18, 0, 0)
    glScalef(0.6, 2.5, 0.4)
    glutSolidCube(12)
    glPopMatrix()
    
    # Cockpit
    glColor3f(0.2, 0.2, 1)
    gluSphere(gluNewQuadric(),8, 12, 12)
    
    glPopMatrix()
def draw_boss_enemy(x, y, z, health):
    glPushMatrix()
    glTranslatef(x, y, z)
    
    # Purple boss with health indication - larger and more intimidating
    health_ratio = health / 150.0
    glColor3f(0.5 + 0.5 * health_ratio, 0, 0.5 + 0.5 * health_ratio)
    gluSphere(gluNewQuadric(),45, 24, 24)  # Larger boss
    
    # Rotating spikes - more of them
    glColor3f(1, 0, 1)
    for i in range(12):  # More spikes
        angle = i * 30 + earth_angle * 3
        glPushMatrix()
        glRotatef(angle, 0, 0, 1)
        glTranslatef(55, 0, 0)
        glScalef(2.5, 0.6, 0.6)
        glutSolidCube(12)
        glPopMatrix()
    
    # Inner rotating core
    glColor3f(1, 0.5, 1)
    glPushMatrix()
    glRotatef(0, 0, 0, 1)
    gluSphere(gluNewQuadric(),20, 16, 16)
    glPopMatrix()
    
    glPopMatrix()
def update_cheat_mode():
    global gun_angle, last_auto_fire_frame, frame_counter
    
    if not cheat_mode:
        return
    
    closest = find_closest_enemy()
    
    if closest:
        # Auto-aim at closest enemy - improved targeting
        dx = closest['x'] - player_pos[0]
        dy = closest['y'] - player_pos[1]
        target_angle = math.degrees(math.atan2(dy, dx))  # angle in degrees to the enemy in XY-plane

        # Smooth rotation with faster response
        angle_diff = (target_angle - gun_angle + 180) % 360 - 180
        gun_angle += angle_diff * 0.2  # ship rotation speed during cheat mode
        
        # MODIFIED: Auto-fire when reasonably aligned, with reduced firing speed
        if abs(angle_diff) < 15 and frame_counter - last_auto_fire_frame > 25: 
            shoot_bullets()
            last_auto_fire_frame = frame_counter
    else:
        # Spin when no enemies - slower
        gun_angle += 1
def shoot_bullets():
    global bullets
    
    # Calculate bullet spawn position
    bx = player_pos[0] + 1 * math.cos(math.radians(gun_angle))
    by = player_pos[1] + 1 * math.sin(math.radians(gun_angle))
    bz = 0 # Ensure bullets at same level
    
    # Weapon level determines firing pattern
    if weapon_level == 1:
        # Single shot
        bullets.append([bx, by, bz, gun_angle, 30]) 
    elif weapon_level == 2:
        # Triple shot
        bullets.append([bx, by, bz, gun_angle, 15])
        bullets.append([bx, by, bz, gun_angle + 12, 15])
        bullets.append([bx, by, bz, gun_angle - 12, 15])
    elif weapon_level == 3:
        # Spread shot (5 bullets)
        for i in range(5):
            angle = gun_angle + (i - 2) * 18
            bullets.append([bx, by, bz, angle, 15])
    elif weapon_level == 4:
        # Wide spread (7 bullets)
        for i in range(7):
            angle = gun_angle + (i - 3) * 22
            bullets.append([bx, by, bz, angle, 15])
    else:  # Level 5+
        # 360 burst - devastating
        for i in range(16):
            angle = gun_angle + i * 22.5
            bullets.append([bx, by, bz, angle, 15])
def update_enemies():
    global enemies, explosions, score, player_health, shield_active, game_over, enemy_size

    for e in enemies[:]:
        dx = 0 - e[0]
        dy = 0 - e[1]
        dist = math.sqrt(dx*dx + dy*dy)
        
        #speed depending on level
        if dist > 0:
            speed = 0.8 + level * 0.05
            e[0] += (dx / dist) * speed
            e[1] += (dy / dist) * speed
        #damage
        #earth
        if dist < 100:
            enemies.remove(e)
            explosions.append([e[0], e[1], 0, e[3] * 1.5, 25])
            player_health -= 0.01*enemy_size
            continue
        #player
        player_dist = math.sqrt((e[0] - player_pos[0])**2 + (e[1] - player_pos[1])**2)
        if player_dist < e[3] + 25:
            enemies.remove(e)
            explosions.append([e[0], e[1], 0, e[3] * 1.5, 25])
            if not shield_active:
                player_health -= 5
                if player_health <= 0:
                    game_over = True
#enemy AI - damage system
def update_enemy_spaceships():
    global enemy_spaceships, enemy_bullets, explosions, player_health, shield_active, game_over, frame_counter
    
    for e in enemy_spaceships[:]:
        dx = player_pos[0] - e[0]
        dy = player_pos[1] - e[1]
        dist = math.sqrt(dx*dx + dy*dy)
        
        if dist > 0:
            speed = 1.0 + level * 0.05
            e[0] += (dx / dist) * speed
            e[1] += (dy / dist) * speed
        
        if dist < 300 and frame_counter - e[4] > 180:
            angle = math.degrees(math.atan2(dy, dx))
            enemy_bullets.append([e[0], e[1], 0, angle, 4])
            e[4] = frame_counter
        
        if dist < 30:
            enemy_spaceships.remove(e)
            explosions.append([e[0], e[1], 0, 35, 30])
            if not shield_active:
                player_health -= 8
                if player_health <= 0:
                    game_over = True
def update_boss_enemies():
    global boss_enemies, enemy_bullets, frame_counter
    
    for b in boss_enemies[:]:
        b[5] += 1
        
        if b[5] % 360 < 180:
            angle = b[5] * 1
            b[0] = 250 * math.cos(math.radians(angle))
            b[1] = 250 * math.sin(math.radians(angle))
        else:
            dx = player_pos[0] - b[0]
            dy = player_pos[1] - b[1]
            dist = math.sqrt(dx*dx + dy*dy)
            if dist > 0:
                speed = 0.3
                b[0] += (dx / dist) * speed
                b[1] += (dy / dist) * speed
        
        if frame_counter - b[4] > 150:
            base_angle = math.degrees(math.atan2(player_pos[1] - b[1], player_pos[0] - b[0]))
            for i in range(5):
                angle = base_angle + (i - 2) * 25
                enemy_bullets.append([b[0], b[1], 0, angle, 4])
            b[4] = frame_counter
def update_enemy_bullets():
    global enemy_bullets, player_health, shield_active, game_over
    
    for b in enemy_bullets[:]:
        b[0] += b[4] * math.cos(math.radians(b[3]))
        b[1] += b[4] * math.sin(math.radians(b[3]))
        
        dist = math.sqrt((b[0] - player_pos[0])**2 + (b[1] - player_pos[1])**2)
        if dist < 25:
            enemy_bullets.remove(b)
            if not shield_active:
                player_health -= 1
                if player_health <= 0:
                    game_over = True
            continue
        
        if abs(b[0]) > GRID_LENGTH + 100 or abs(b[1]) > GRID_LENGTH + 100:
            enemy_bullets.remove(b)

#look in draw_shapes(() for weapons upgrades(multiple bullets)
#################################################################################


#Sakib's part
##################################################################################
def draw_resource(x, y, z, rotation):
    # Golden crystal - made more visible and attractive
    glPushMatrix()
    glTranslatef(x, y, z)
    glRotatef(rotation, 0, 0, 1)
    glRotatef(rotation * 0.7, 1, 0, 0)
    
    # Bright golden color
    glColor3f(1, 0.9, 0)
    
    # Draw octahedron using two pyramids - larger
    glPushMatrix()
    glScalef(18, 18, 18)  # Larger size
    # Top pyramid
    glBegin(GL_TRIANGLES)
    glVertex3f(1, 0, 0)
    glVertex3f(0, 1, 0)
    glVertex3f(0, 0, 1)
    
    glVertex3f(0, 1, 0)
    glVertex3f(-1, 0, 0)
    glVertex3f(0, 0, 1)
    
    glVertex3f(-1, 0, 0)
    glVertex3f(0, -1, 0)
    glVertex3f(0, 0, 1)
    
    glVertex3f(0, -1, 0)
    glVertex3f(1, 0, 0)
    glVertex3f(0, 0, 1)
    
    # Bottom pyramid
    glVertex3f(1, 0, 0)
    glVertex3f(0, 1, 0)
    glVertex3f(0, 0, -1)
    
    glVertex3f(0, 1, 0)
    glVertex3f(-1, 0, 0)
    glVertex3f(0, 0, -1)
    
    glVertex3f(-1, 0, 0)
    glVertex3f(0, -1, 0)
    glVertex3f(0, 0, -1)
    
    glVertex3f(0, -1, 0)
    glVertex3f(1, 0, 0)
    glVertex3f(0, 0, -1)
    glEnd()
    glPopMatrix()
    
    # Add glowing effect
    glColor3f(1, 1, 0.5)
    gluSphere(gluNewQuadric(),12, 12, 12)
    
    glPopMatrix()
def find_closest_enemy():
    min_dist = float('inf')
    closest = None
    
    # Check all enemy types
    for e in enemies:
        dx = e[0] - player_pos[0]
        dy = e[1] - player_pos[1]
        dist = dx*dx + dy*dy  #x^2+y^2
        if dist < min_dist:   #loop to update closest enemy
            min_dist = dist
            closest = {'x': e[0], 'y': e[1], 'type': 'asteroid'}
    
    for e in enemy_spaceships:
        dx = e[0] - player_pos[0]
        dy = e[1] - player_pos[1]
        dist = dx*dx + dy*dy
        if dist < min_dist:
            min_dist = dist
            closest = {'x': e[0], 'y': e[1], 'type': 'ship'}
    
    for b in boss_enemies:
        dx = b[0] - player_pos[0]
        dy = b[1] - player_pos[1]
        dist = dx*dx + dy*dy
        if dist < min_dist:
            min_dist = dist
            closest = {'x': b[0], 'y': b[1], 'type': 'boss'}
    
    return closest
#score 
def update_bullets():
    global bullets, enemies, enemy_spaceships, boss_enemies, score, explosions, missed_bullets, score_multiplier
    
    for b in bullets[:]:
        b[0] += b[4] * math.cos(math.radians(b[3]))
        b[1] += b[4] * math.sin(math.radians(b[3]))
        
        hit = False
        
        for e in enemies[:]:
            dist = math.sqrt((e[0] - b[0])**2 + (e[1] - b[1])**2)
            if dist < e[3] + 10:
                enemies.remove(e)
                bullets.remove(b)
                explosions.append([e[0], e[1], 0, e[3] * 1.5, 25])
                score += 1 * score_multiplier
                hit = True
                break
        
        if hit:
            continue
        
        for e in enemy_spaceships[:]:
            dist = math.sqrt((e[0] - b[0])**2 + (e[1] - b[1])**2)
            if dist < 25:
                e[3] -= 8
                bullets.remove(b)
                if e[3] <= 0:
                    enemy_spaceships.remove(e)
                    explosions.append([e[0], e[1], 0, 35, 30])
                    score += 3 * score_multiplier
                hit = True
                break
        
        if hit:
            continue
        
        for boss in boss_enemies[:]:
            dist = math.sqrt((boss[0] - b[0])**2 + (boss[1] - b[1])**2)
            if dist < 50:
                boss[3] -= 3
                bullets.remove(b)
                if boss[3] <= 0:
                    boss_enemies.remove(boss)
                    explosions.append([boss[0], boss[1], 0, 80, 40])
                    score += 25 * score_multiplier
                hit = True
                break
        
        if hit:
            continue
        
        if abs(b[0]) > GRID_LENGTH + 100 or abs(b[1]) > GRID_LENGTH + 100:
            bullets.remove(b)
            missed_bullets += 1
#score multiplier
def update_resources():
    global resources, score, total_points, score_multiplier, frame_counter

    # MODIFIED: Remove resources that have expired (older than 5 seconds)
    resources[:] = [r for r in resources if frame_counter - r[4] <= RESOURCE_LIFETIME_FRAMES]
    
    for r in resources[:]:
        r[3] += 5
        
        dist = math.sqrt((r[0] - player_pos[0])**2 + (r[1] - player_pos[1])**2)
        if dist < 35:
            resources.remove(r)
            points = 8 * score_multiplier
            score += points

##################################################################################


#main rendering and control functions
####################################################################################
def draw_shapes():
    global moon_angle, earth_angle
    
    # Draw 800 stars background 
    glPointSize(1.5)
    glBegin(GL_POINTS)
    glColor3f(1.0, 1.0, 1.0)
    random.seed(42)
    for i in range(800):
        x = random.uniform(-2000, 2000)
        y = random.uniform(-2000, 2000)
        z = random.uniform(-800, 500)
        brightness = random.uniform(0.3, 1.0)
        glColor3f(brightness, brightness, brightness)
        glVertex3f(x, y, z)
    random.seed() #better randomization
    glEnd()
    
    # Draw Earth - larger and more detailed
    glPushMatrix()
    glTranslatef(0, 0, -200)
    glRotatef(earth_angle, 0, 0, 1)
    glColor3f(0, 0.4, 0.9)
    gluSphere(gluNewQuadric(),180, 32, 32)
    glPopMatrix()
    
    # Draw Moon - more realistic orbit
    moon_x = 350 * math.cos(math.radians(moon_angle)) #orbit radius = 350
    moon_y = 350 * math.sin(math.radians(moon_angle))
    glPushMatrix()
    glTranslatef(moon_x, moon_y, -120)
    glColor3f(0.8, 0.8, 0.7)
    gluSphere(gluNewQuadric(),45, 20, 20)
    glPopMatrix()
    
    # Draw healing center - ENHANCED VISIBILITY
    draw_healing_center()
    
    # Draw spaceship - improved design
    glPushMatrix()
    glTranslatef(player_pos[0], player_pos[1], player_pos[2])
    glRotatef(gun_angle, 0, 0, 1)
    
    # Shield effect if active - larger and more visible
    if shield_active:
        glColor3f(0, 0.6, 1)
        gluSphere(gluNewQuadric(),50, 16, 16)
    
    # Cheat mode indicator - more prominent
    if cheat_mode:
        glColor3f(1, 0, 0)
        glPushMatrix()
        glTranslatef(0, 0, 35)
        gluSphere(gluNewQuadric(),10, 12, 12)
        glPopMatrix()
        # Additional cheat indicator
        glColor3f(1, 0.5, 0)
        for i in range(4):
            glPushMatrix()
            glRotatef(i * 90 + earth_angle * 5, 0, 0, 1)
            glTranslatef(40, 0, 20)
            gluSphere(gluNewQuadric(),5, 8, 8)
            glPopMatrix()
    
    # Main hull - improved appearance
    glColor3f(0.8, 0.8, 0.95)
    glPushMatrix()
    glScalef(1.8, 0.9, 0.6)
    gluSphere(gluNewQuadric(),22, 20, 20)
    glPopMatrix()
    
    # Cockpit - more prominent
    glColor3f(0.2, 0.7, 1.0)
    glPushMatrix()
    glTranslatef(15, 0, 8)
    gluSphere(gluNewQuadric(),12, 14, 14)
    glPopMatrix()
    
    # Weapon level indicators - more visible
    if weapon_level > 1:
        glColor3f(1, 1, 0)
        for i in range(weapon_level - 1):
            glPushMatrix()
            glTranslatef(20, (i - (weapon_level-2)/2) * 12, 5)
            gluSphere(gluNewQuadric(),4, 10, 10)
            glPopMatrix()
    
    # Engine exhaust effect
    glColor3f(0, 0.5, 1)
    glPushMatrix()
    glTranslatef(-25, 0, 0)
    glScalef(0.8, 0.8, 1.5)
    gluSphere(gluNewQuadric(),8, 12, 12)
    glPopMatrix()
    
    glPopMatrix()
    
    # Draw enemies - all at EXACT same Z level 0
    for e in enemies:
        glPushMatrix()
        glTranslatef(e[0], e[1], 0)  # Force exact Z level
        glRotatef(moon_angle * 2, 1, 0.5, 0.3)
        # Different colors for variety
        size_factor = e[3] / 20.0
        glColor3f(0.6 + 0.2 * size_factor, 0.4, 0.4)
        gluSphere(gluNewQuadric(),e[3], 16, 16)
        glPopMatrix()
    
    # Draw enemy spaceships - exact Z level 0
    for e in enemy_spaceships:
        draw_enemy_spaceship(e[0], e[1], 0, e[3])
    
    # Draw boss enemies - exact Z level
    for b in boss_enemies:
        draw_boss_enemy(b[0], b[1], 0, b[3])
    
    # Draw power-ups - exact Z level
    for p in power_ups:
        draw_power_up(p[0], p[1], 0, p[3], p[4])
    
    # Draw resources - exact Z level
    for r in resources:
        draw_resource(r[0], r[1], 0, r[3])
    
    # Draw explosions - fade in effect
    for exp in explosions:
        glPushMatrix()
        glTranslatef(exp[0], exp[1], 0)
        # Colorful explosion
        fade = exp[4] / 30.0
        glColor3f(1.0, 0.5 * fade, 0.0)
        gluSphere(gluNewQuadric(),exp[3], 12, 12)
        # Inner bright core
        glColor3f(1.0, 1.0, 0.5)
        gluSphere(gluNewQuadric(),exp[3] * 0.5, 8, 8)
        glPopMatrix()
    
    # Draw bullets - exact Z level
    glColor3f(1, 1, 0)
    for b in bullets:
        glPushMatrix()
        glTranslatef(b[0], b[1], 0)
        gluSphere(gluNewQuadric(),6, 10, 10)
        glPopMatrix()
    
    # Draw enemy bullets - exact Z level
    glColor3f(1, 0.2, 0.2)
    for b in enemy_bullets:
        glPushMatrix()
        glTranslatef(b[0], b[1], 0)
        gluSphere(gluNewQuadric(),5, 8, 8)
        glPopMatrix()
    
    # Draw grid - enhanced visibility
    glColor3f(0, 0.6, 0.6)
    glBegin(GL_LINES)
    for i in range(-GRID_LENGTH, GRID_LENGTH + 100, 100): #each 100 unites from origin
        glVertex3f(i, -GRID_LENGTH, 0)
        glVertex3f(i, GRID_LENGTH, 0)
        glVertex3f(-GRID_LENGTH, i, 0)
        glVertex3f(GRID_LENGTH, i, 0)
    glEnd()
#ship movement, cheat mode 
def keyboardListener(key, x, y):
    global gun_angle, player_pos, game_over, score, total_points, cheat_mode, player_health, weapon_level, speed_boost_active, speed_boost_end_frame, shield_active, shield_end_frame, frame_counter, first_person_view, enemies, enemy_spaceships, boss_enemies, bullets, enemy_bullets, power_ups, resources, explosions, level, last_level_up_score, missed_bullets, last_manual_fire_frame
    
    if key == b'r' and game_over:
        # Reset game but keep total points
        total_points += score
        score = 0
        game_over = False
        player_pos = [0, 200, 0]  # Start closer to healing center
        gun_angle = 0
        player_health = player_max_health
        weapon_level = 1
        level = 1
        last_level_up_score = 0
        speed_boost_active = False
        shield_active = False
        cheat_mode = False
        missed_bullets = 0
        
        # Clear all lists
        enemies.clear()
        enemy_spaceships.clear()
        boss_enemies.clear()
        bullets.clear()
        enemy_bullets.clear()
        power_ups.clear()
        resources.clear()
        explosions.clear()
        return
    if game_over:
        return
    
    # Speed and boost
    speed = 5
    if speed_boost_active:
        speed *= 5  
    
    if key == b'w' or key == b'W':
        # Move forward
        new_x = player_pos[0] + speed * math.cos(math.radians(gun_angle))
        new_y = player_pos[1] + speed * math.sin(math.radians(gun_angle))
        # Keep within bounds
        if abs(new_x) < GRID_LENGTH - 50 and abs(new_y) < GRID_LENGTH - 50:
            player_pos[0] = new_x
            player_pos[1] = new_y
    elif key == b's' or key == b'S':
        # Move backward
        new_x = player_pos[0] - speed * math.cos(math.radians(gun_angle))
        new_y = player_pos[1] - speed * math.sin(math.radians(gun_angle))
        # Keep within bounds
        if abs(new_x) < GRID_LENGTH - 50 and abs(new_y) < GRID_LENGTH - 50:
            player_pos[0] = new_x
            player_pos[1] = new_y
    elif key == b'a' or key == b'A':
        gun_angle += 10  # Slightly faster rotation
    elif key == b'd'or key == b'D':
        gun_angle -= 10
    elif key == b' ':
        # fires every 10 frames
        if frame_counter - last_manual_fire_frame > 10:
            shoot_bullets()
            last_manual_fire_frame = frame_counter
    elif key == b'v' or key == b'V':
        first_person_view = not first_person_view
    elif key == b'c' or key == b'C':
        # Activate speed boost
        if not speed_boost_active:
            speed_boost_active = True
            speed_boost_end_frame = frame_counter + 360  # 6 seconds
    elif key == b'q' or key == b'Q':
        # Toggle cheat mode
        cheat_mode = not cheat_mode
#camera position change
def specialKeyListener(key, x, y):
    global camera_pos
    cx, cy, cz = camera_pos
    if key == GLUT_KEY_LEFT:
        cx -= 15
    elif key == GLUT_KEY_RIGHT:
        cx += 15
    elif key == GLUT_KEY_UP:
        cz += 15
    elif key == GLUT_KEY_DOWN:
        cz -= 15
    elif key == GLUT_KEY_PAGE_UP:
        cy += 15
    elif key == GLUT_KEY_PAGE_DOWN:
        cy -= 15
    camera_pos = (cx, cy, cz)
#shooting
def mouseListener(button, state, x, y):
    global last_manual_fire_frame
    if button == GLUT_LEFT_BUTTON and state == GLUT_DOWN and not game_over:
        if frame_counter - last_manual_fire_frame > 10:
            shoot_bullets()
            last_manual_fire_frame = frame_counter

def spawn_enemy():
    global frame_counter, last_enemy_spawn_frame,last_boss_spawn_frame, level ,enemy_size
    
    # MODIFIED: Increased enemy spawn rate for more action
    spawn_interval = max(90 - level * 5, 50)
    
    if frame_counter - last_enemy_spawn_frame > spawn_interval:
        # Random spawn position at edge of map
        side = random.choice(['left', 'right', 'top', 'bottom'])
        if side == 'left':
            ex = -GRID_LENGTH + 50
            ey = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
        elif side == 'right':
            ex = GRID_LENGTH - 50
            ey = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
        elif side == 'top':
            ex = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
            ey = GRID_LENGTH - 50
        else:
            ex = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
            ey = -GRID_LENGTH + 50
        
        ez = 0
        
        # Spawn different enemy types
        rand = random.random()
        if rand < 0.25 and level > 2:                   #25% chance, if level > 2, spawn stronger enemy spaceship 
            enemy_spaceships.append([ex, ey, ez, 25, 0])
        else:
            enemy_size = random.uniform(12, 20)
            enemies.append([ex, ey, ez, enemy_size, 0])
        
        last_enemy_spawn_frame = frame_counter
        
        # Spawn power-ups if under the limit within central area of grid
        if len(power_ups) < MAX_POWERUPS and random.random() < 0.4:
            px = random.uniform(-GRID_LENGTH//3, GRID_LENGTH//3)
            py = random.uniform(-GRID_LENGTH//3, GRID_LENGTH//3)
            ptype = random.randint(0, 3)
            power_ups.append([px, py, 0, ptype, 0, frame_counter])
    #spawn boss       
    if level >= 3 and level % 3 == 0:
        if frame_counter - last_boss_spawn_frame > 1200:
            if len(boss_enemies) == 0:
                bx = random.uniform(-200, 200)
                by = random.uniform(-200, 200)
                boss_enemies.append([bx, by, 0, 150, 0, 0])
                last_boss_spawn_frame = frame_counter



def manage_resources():
    global resources, frame_counter
    
    # Ensure a minimum number of resources are always available
    if len(resources) < MIN_RESOURCES:
        for _ in range(MIN_RESOURCES - len(resources)):
            rx = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
            ry = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
            resources.append([rx, ry, 0, 0, frame_counter])
    
    # Spawn new resources if below the maximum limit
    if len(resources) < MAX_RESOURCES and random.random() < 0.1: # Chance to spawn new ones gradually
        rx = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
        ry = random.uniform(-GRID_LENGTH//2, GRID_LENGTH//2)
        resources.append([rx, ry, 0, 0, frame_counter])

def update_explosions():
    global explosions
    
    for exp in explosions[:]:
        exp[4] -= 1
        exp[3] *= 0.92
        if exp[4] <= 0:
            explosions.remove(exp)

def idle():
    global moon_angle, earth_angle, frame_counter, player_health, level, last_level_up_score, score_multiplier, speed_boost_active, speed_boost_end_frame, shield_active, shield_end_frame
    
    if game_over:
        glutPostRedisplay()
        return
    
    frame_counter += 1
    
    moon_angle += 0.4
    earth_angle += 0.2
    
    update_cheat_mode()
    
    if speed_boost_active and frame_counter > speed_boost_end_frame:
        speed_boost_active = False
    
    if shield_active and frame_counter > shield_end_frame:
        shield_active = False
    
    if score - last_level_up_score >= 15:
        level += 1
        last_level_up_score = score
        score_multiplier = level
        player_health = min(player_max_health, player_health + 20)
    
    dist_to_center = math.sqrt(player_pos[0]**2 + player_pos[1]**2)
    if dist_to_center < 120:
        player_health = min(player_max_health, player_health + 1.2)
    
    spawn_enemy()
    
    # MODIFIED: Call the new resource manager
    manage_resources()
    
    update_enemies()
    update_enemy_spaceships()
    update_boss_enemies()
    update_bullets()
    update_enemy_bullets()
    update_power_ups()
    update_resources()
    update_explosions()
    
    glutPostRedisplay()
    time.sleep(0.02)

def showScreen():
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
    glClearColor(0.0, 0.0, 0.15, 1.0)
    glLoadIdentity()
    
    glViewport(0, 0, WINDOW_WIDTH, WINDOW_HEIGHT)
    setupCamera()
    
    glEnable(GL_DEPTH_TEST)
    
    draw_shapes()
    draw_health_bar()
    draw_mini_map()
    
    
#Promit-Score/info display
###########################################################################################################
    glColor3f(1, 1, 0.8)
    draw_text(15, WINDOW_HEIGHT - 35, f"SCORE: {score}  |  LEVEL: {level}  |  MULTIPLIER: x{score_multiplier}", GLUT_BITMAP_TIMES_ROMAN_24)
    
    glColor3f(1, 1, 0)
    draw_text(15, WINDOW_HEIGHT - 65, f"*** TOTAL CAREER POINTS: {total_points + score} ***", GLUT_BITMAP_TIMES_ROMAN_24)
    
    glColor3f(0.8, 1, 0.8)
    draw_text(15, WINDOW_HEIGHT - 90, f"HEALTH: {int(player_health)}/{player_max_health}  |  WEAPON LEVEL: {weapon_level}")
    
    glColor3f(0.8, 0.8, 1)
    effects_text = f"CHEAT: {'ON' if cheat_mode else 'OFF'} (S key)"
    if speed_boost_active:
        effects_text += "  |  SPEED BOOST ACTIVE"
    if shield_active:
        effects_text += "  |  SHIELD ACTIVE"
    draw_text(15, WINDOW_HEIGHT - 115, effects_text)
    
    glColor3f(1, 0.8, 1)
    draw_text(15, WINDOW_HEIGHT - 140, f"OBJECTS: {len(resources)}/{MAX_RESOURCES} Crystals  |  {len(power_ups)}/{MAX_POWERUPS} Power-ups  |  {len(enemies) + len(enemy_spaceships)} Enemies")
    
    if len(boss_enemies) > 0:
        glColor3f(1, 0, 0)
        boss_health = boss_enemies[0][3]
        draw_text(WINDOW_WIDTH // 2 - 150, WINDOW_HEIGHT - 80, f"!!! BOSS BATTLE - HEALTH: {boss_health} !!!", GLUT_BITMAP_TIMES_ROMAN_24)
    
    glColor3f(0.9, 0.9, 0.9)
    draw_text(15, 50, "CONTROLS: W/Q=Move  A/D=Rotate  SPACE/Click=Shoot  C=Speed  S=Cheat  V=Camera  R=Reset")
    draw_text(15, 30, "HEALING ZONE: Stay near the GREEN CENTER to heal continuously!")
    draw_text(15, 10, "COLLECT: Yellow power-ups for weapons, Red for health, Blue for speed, Purple for shield (15s limit)")
    
    if weapon_level > 1:
        glColor3f(1, 1, 0)
        weapon_patterns = ["Single", "Triple", "Spread-5", "Wide-7", "360-Burst"]
        pattern = weapon_patterns[min(weapon_level - 1, 4)]
        draw_text(WINDOW_WIDTH - 300, WINDOW_HEIGHT - 35, f"WEAPON: {pattern}", GLUT_BITMAP_TIMES_ROMAN_24)
    
    dist_to_heal = math.sqrt(player_pos[0]**2 + player_pos[1]**2)
    if dist_to_heal > 120:
        glColor3f(0, 1, 0)
        draw_text(WINDOW_WIDTH // 2 - 100, 80, f"HEALING CENTER: {int(dist_to_heal)} units away")
    else:
        glColor3f(0, 1, 0.5)
        draw_text(WINDOW_WIDTH // 2 - 80, 80, "*** HEALING ACTIVE ***")
    
    if game_over:
        glColor3f(1, 0.2, 0.2)
        draw_text(WINDOW_WIDTH // 2 - 100, WINDOW_HEIGHT // 2 + 80, "GAME OVER", GLUT_BITMAP_TIMES_ROMAN_24)
        
        glColor3f(1, 1, 1)
        draw_text(WINDOW_WIDTH // 2 - 120, WINDOW_HEIGHT // 2 + 40, f"FINAL SCORE: {score}")
        draw_text(WINDOW_WIDTH // 2 - 120, WINDOW_HEIGHT // 2 + 10, f"LEVEL REACHED: {level}")
        
        glColor3f(1, 1, 0)
        draw_text(WINDOW_WIDTH // 2 - 180, WINDOW_HEIGHT // 2 - 20, f"TOTAL CAREER POINTS: {total_points + score}", GLUT_BITMAP_TIMES_ROMAN_24)
        
        glColor3f(0.8, 1, 0.8)
        draw_text(WINDOW_WIDTH // 2 - 100, WINDOW_HEIGHT // 2 - 60, "Press R to Restart")
        
        accuracy = 0 if (score + missed_bullets) == 0 else score / (score + missed_bullets) * 100
        draw_text(WINDOW_WIDTH // 2 - 100, WINDOW_HEIGHT // 2 - 100, f"Accuracy: {accuracy:.1f}%")
##########################################################################################################################################    
    glutSwapBuffers()

def main():
    glutInit()
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB | GLUT_DEPTH)
    glutInitWindowSize(WINDOW_WIDTH, WINDOW_HEIGHT)
    glutInitWindowPosition(0, 0)
    glutCreateWindow(b"Guardian of the Galaxy")
    
    glutDisplayFunc(showScreen)
    glutKeyboardFunc(keyboardListener)
    glutSpecialFunc(specialKeyListener)
    glutMouseFunc(mouseListener)
    glutIdleFunc(idle)
    
    glutMainLoop()

if __name__ == "__main__":
    main()
