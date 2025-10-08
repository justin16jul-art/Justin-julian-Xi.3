import pygame
import sys
import random

# --- 1. Inisialisasi Pygame dan Aset ---
pygame.init()
pygame.mixer.init()

# Konstanta Layar
SCREEN_WIDTH = 500
SCREEN_HEIGHT = 800
SCREEN = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption('Flappy Bird')
CLOCK = pygame.time.Clock()
FPS = 120
FONT = pygame.font.Font('freesansbold.ttf', 40)

# Variabel Game
GRAVITY = 0.25
BIRD_MOVEMENT = 0
GAME_ACTIVE = True
SCORE = 0
HIGH_SCORE = 0

# Aset Gambar (Pastikan file ada di folder 'assets/')
try:
    BG_SURFACE = pygame.transform.scale(pygame.image.load('assets/background.png').convert(), (SCREEN_WIDTH, SCREEN_HEIGHT))
    FLOOR_SURFACE = pygame.transform.scale2x(pygame.image.load('assets/base.png').convert())
    
    BIRD_DOWN = pygame.transform.scale2x(pygame.image.load('assets/bluebird-downflap.png').convert_alpha())
    BIRD_MID = pygame.transform.scale2x(pygame.image.load('assets/bluebird-midflap.png').convert_alpha())
    BIRD_UP = pygame.transform.scale2x(pygame.image.load('assets/bluebird-upflap.png').convert_alpha())
    BIRD_FRAMES = [BIRD_DOWN, BIRD_MID, BIRD_UP]
    
    PIPE_SURFACE = pygame.transform.scale2x(pygame.image.load('assets/pipe-green.png').convert_alpha())
    GAME_OVER_SURFACE = pygame.transform.scale2x(pygame.image.load('assets/gameover.png').convert_alpha())
except pygame.error as e:
    print(f"Error memuat aset: {e}. Pastikan Anda memiliki folder 'assets' yang berisi semua gambar.")
    sys.exit()

# Aset Suara
try:
    FLAP_SOUND = pygame.mixer.Sound('assets/sound/wing.wav')
    HIT_SOUND = pygame.mixer.Sound('assets/sound/hit.wav')
    SCORE_SOUND = pygame.mixer.Sound('assets/sound/point.wav')
except pygame.error as e:
    print(f"Error memuat suara: {e}. Pastikan Anda memiliki folder 'assets/sound' yang berisi semua file .wav.")
    # Lanjutkan tanpa suara jika ada error, agar game tetap bisa berjalan.
    FLAP_SOUND = HIT_SOUND = SCORE_SOUND = None

# Inisialisasi Burung
bird_index = 0
BIRD_SURFACE = BIRD_FRAMES[bird_index]
BIRD_RECT = BIRD_SURFACE.get_rect(center = (100, SCREEN_HEIGHT // 2))

# Inisialisasi Pipa
PIPE_LIST = []
PIPE_HEIGHT = [400, 600, 700] 

# Timers Kustom
SPAWNPIPE = pygame.USEREVENT
pygame.time.set_timer(SPAWNPIPE, 1200) # Spawn pipa setiap 1.2 detik
BIRDFLAP = pygame.USEREVENT + 1
pygame.time.set_timer(BIRDFLAP, 200) # Animasi kepakan sayap

floor_x_pos = 0
score_sound_cooldown = 0

# --- 2. Fungsi-fungsi Game ---

def draw_floor():
    """Menggambar lantai yang bergerak tanpa batas."""
    global floor_x_pos
    SCREEN.blit(FLOOR_SURFACE, (floor_x_pos, 700))
    SCREEN.blit(FLOOR_SURFACE, (floor_x_pos + SCREEN_WIDTH, 700))
    floor_x_pos -= 1
    if floor_x_pos <= -SCREEN_WIDTH:
        floor_x_pos = 0

def create_pipe():
    """Membuat pasangan pipa dengan ketinggian acak."""
    random_pipe_pos = random.choice(PIPE_HEIGHT)
    # Gap antar pipa adalah 150 piksel
    bottom_pipe = PIPE_SURFACE.get_rect(midtop = (SCREEN_WIDTH + 100, random_pipe_pos))
    top_pipe = PIPE_SURFACE.get_rect(midbottom = (SCREEN_WIDTH + 100, random_pipe_pos - 150))
    return bottom_pipe, top_pipe

def move_pipes(pipes):
    """Menggerakkan pipa-pipa ke kiri."""
    for pipe in pipes:
        pipe.centerx -= 5
    # Menghapus pipa yang sudah tidak terlihat
    return [pipe for pipe in pipes if pipe.right > -50]

def draw_pipes(pipes):
    """Menggambar semua pipa."""
    for pipe in pipes:
        if pipe.bottom >= SCREEN_HEIGHT:
            SCREEN.blit(PIPE_SURFACE, pipe)
        else:
            # Pipa atas dibalik
            flip_pipe = pygame.transform.flip(PIPE_SURFACE, False, True)
            SCREEN.blit(flip_pipe, pipe)

def check_collision(pipes):
    """Mengecek tabrakan burung dengan pipa atau batas layar."""
    # Cek tabrakan dengan pipa
    for pipe in pipes:
        if BIRD_RECT.colliderect(pipe):
            if HIT_SOUND: HIT_SOUND.play() # 💥 Bunyi tabrakan
            return False
    
    # Cek tabrakan dengan lantai (700) atau langit-langit (0)
    if BIRD_RECT.top <= 0 or BIRD_RECT.bottom >= 700:
        if HIT_SOUND: HIT_SOUND.play() # 💥 Bunyi tabrakan
        return False
        
    return True

def rotate_bird(bird):
    """Membuat efek burung menukik berdasarkan kecepatan vertikal."""
    return pygame.transform.rotozoom(bird, -BIRD_MOVEMENT * 3, 1)

def bird_animation():
    """Mengubah frame untuk animasi kepakan sayap."""
    new_bird = BIRD_FRAMES[bird_index]
    new_bird_rect = new_bird.get_rect(center = (100, BIRD_RECT.centery))
    return new_bird, new_bird_rect

def score_display(game_state):
    """Menampilkan skor dan high score."""
    if game_state == 'main_game':
        score_surface = FONT.render(str(int(SCORE)), True, (255, 255, 255))
        score_rect = score_surface.get_rect(center = (SCREEN_WIDTH // 2, 50))
        SCREEN.blit(score_surface, score_rect)
    
    if game_state == 'game_over':
        # Skor saat ini
        score_surface = FONT.render(f'Score: {int(SCORE)}', True, (255, 255, 255))
        score_rect = score_surface.get_rect(center = (SCREEN_WIDTH // 2, 450))
        SCREEN.blit(score_surface, score_rect)
        
        # High Score
        high_score_surface = FONT.render(f'High Score: {int(HIGH_SCORE)}', True, (255, 255, 255))
        high_score_rect = high_score_surface.get_rect(center = (SCREEN_WIDTH // 2, 500))
        SCREEN.blit(high_score_surface, high_score_rect)

def update_score(score, high_score):
    """Memperbarui High Score."""
    if score > high_score:
        high_score = score
    return high_score

# --- 3. Game Loop Utama ---

while True:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
            
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_SPACE:
                if GAME_ACTIVE:
                    # 🚀 Burung mengepak (flap)
                    BIRD_MOVEMENT = 0
                    BIRD_MOVEMENT -= 10
                    if FLAP_SOUND: FLAP_SOUND.play() # 🎵 Bunyi sayap
                else:
                    # 🔄 Restart Game
                    GAME_ACTIVE = True
                    PIPE_LIST.clear()
                    BIRD_RECT.center = (100, SCREEN_HEIGHT // 2)
                    BIRD_MOVEMENT = 0
                    SCORE = 0

        # Event untuk timer kustom
        if event.type == SPAWNPIPE and GAME_ACTIVE:
            PIPE_LIST.extend(create_pipe())

        if event.type == BIRDFLAP:
            # Animasi sayap
            bird_index = (bird_index + 1) % 3
            BIRD_SURFACE, BIRD_RECT = bird_animation()

    # Menggambar Background
    SCREEN.blit(BG_SURFACE, (0, 0))

    if GAME_ACTIVE:
        # Gerakan Burung (Gravitasi)
        BIRD_MOVEMENT += GRAVITY
        rotated_bird = rotate_bird(BIRD_SURFACE)
        BIRD_RECT.centery += BIRD_MOVEMENT
        SCREEN.blit(rotated_bird, BIRD_RECT)

        # Cek Tabrakan -> SET Game Over
        GAME_ACTIVE = check_collision(PIPE_LIST)
        
        # Pipa
        PIPE_LIST = move_pipes(PIPE_LIST)
        draw_pipes(PIPE_LIST)

        # Update Skor dan Bunyi Poin
        for pipe in PIPE_LIST:
            # Hitungan skor hanya untuk pipa di sebelah kanan burung
            if pipe.right < 100 and pipe.left > 95:
                # Skor bertambah 1 per pasangan pipa yang dilewati (0.5 + 0.5)
                SCORE += 0.5
                
                # Cooldown untuk bunyi agar tidak berulang
                if SCORE_SOUND and score_sound_cooldown <= 0:
                    SCORE_SOUND.play()
                    score_sound_cooldown = 100 

        score_sound_cooldown -= 1
        score_display('main_game')

    else:
        # Layar Game Over 💀
        SCREEN.blit(GAME_OVER_SURFACE, (SCREEN_WIDTH // 2 - 192 // 2, 300)) # Pusat Game Over Image
        HIGH_SCORE = update_score(SCORE, HIGH_SCORE)
        score_display('game_over')
    
    # Menggambar Lantai (selalu di atas background dan di bawah Game Over)
    draw_floor()

    # Update Layar dan FPS
    pygame.display.update()
    CLOCK.tick(FPS)
    
