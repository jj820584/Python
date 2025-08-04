# Python
## 파이썬으로 벽돌개기 게임 만들기



<img width="803" height="630" alt="image" src="https://github.com/user-attachments/assets/3915ad11-e473-4a41-b8d8-cad3580b4cb4" />


```c
import pygame
import sys
import random
import math

# 게임 초기화
pygame.init()

# 색상 정의
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BLUE = (0, 0, 255)
ORANGE = (255, 165, 0)
YELLOW = (255, 255, 0)
PURPLE = (128, 0, 128)
PINK = (255, 192, 203)
CYAN = (0, 255, 255)
LIGHT_BLUE = (173, 216, 230)
GOLD = (255, 215, 0)

# 게임 설정
SCREEN_WIDTH = 800
SCREEN_HEIGHT = 600
PADDLE_WIDTH = 100
PADDLE_HEIGHT = 15
BALL_SIZE = 15
BRICK_WIDTH = 75
BRICK_HEIGHT = 30
BRICK_ROWS = 6
BRICK_COLS = 10

class Ball:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.dx = random.choice([-5, 5])
        self.dy = -5
        self.size = BALL_SIZE
        self.speed_multiplier = 1.0
    
    def move(self):
        self.x += self.dx * self.speed_multiplier
        self.y += self.dy * self.speed_multiplier
    
    def bounce_x(self):
        self.dx = -self.dx
    
    def bounce_y(self):
        self.dy = -self.dy
    
    def draw(self, screen):
        pygame.draw.circle(screen, WHITE, (int(self.x), int(self.y)), self.size)

class Paddle:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.width = PADDLE_WIDTH
        self.height = PADDLE_HEIGHT
        self.speed = 8
        self.original_width = PADDLE_WIDTH
    
    def move_left(self):
        if self.x > 0:
            self.x -= self.speed
    
    def move_right(self):
        if self.x < SCREEN_WIDTH - self.width:
            self.x += self.speed
    
    def draw(self, screen):
        pygame.draw.rect(screen, WHITE, (self.x, self.y, self.width, self.height))

class Brick:
    def __init__(self, x, y, color, has_item=False):
        self.x = x
        self.y = y
        self.width = BRICK_WIDTH
        self.height = BRICK_HEIGHT
        self.color = color
        self.destroyed = False
        self.has_item = has_item
    
    def draw(self, screen):
        if not self.destroyed:
            pygame.draw.rect(screen, self.color, (self.x, self.y, self.width, self.height))
            pygame.draw.rect(screen, BLACK, (self.x, self.y, self.width, self.height), 2)
            
            # 아이템이 있는 벽돌은 별표시
            if self.has_item:
                star_x = self.x + self.width // 2
                star_y = self.y + self.height // 2
                self.draw_star(screen, star_x, star_y, 8, GOLD)
    
    def draw_star(self, screen, x, y, size, color):
        points = []
        for i in range(10):
            angle = math.pi * i / 5
            if i % 2 == 0:
                radius = size
            else:
                radius = size // 2
            px = x + radius * math.cos(angle - math.pi / 2)
            py = y + radius * math.sin(angle - math.pi / 2)
            points.append((px, py))
        pygame.draw.polygon(screen, color, points)

class Item:
    def __init__(self, x, y, item_type):
        self.x = x
        self.y = y
        self.item_type = item_type
        self.dy = 3
        self.width = 30
        self.height = 20
        self.collected = False
        
        # 아이템별 색상
        self.colors = {
            'expand_paddle': GREEN,
            'shrink_paddle': RED,
            'multi_ball': BLUE,
            'extra_life': PINK,
            'slow_ball': CYAN,
            'fast_ball': ORANGE,
            'power_ball': PURPLE
        }
    
    def move(self):
        self.y += self.dy
    
    def draw(self, screen):
        if not self.collected:
            color = self.colors.get(self.item_type, WHITE)
            pygame.draw.rect(screen, color, (self.x, self.y, self.width, self.height))
            pygame.draw.rect(screen, BLACK, (self.x, self.y, self.width, self.height), 2)
            
            # 아이템 아이콘 그리기
            font = pygame.font.Font(None, 16)
            icons = {
                'expand_paddle': 'E',
                'shrink_paddle': 'S',
                'multi_ball': 'M',
                'extra_life': '+',
                'slow_ball': 'SL',
                'fast_ball': 'F',
                'power_ball': 'P'
            }
            icon = icons.get(self.item_type, '?')
            text = font.render(icon, True, BLACK)
            text_rect = text.get_rect(center=(self.x + self.width//2, self.y + self.height//2))
            screen.blit(text, text_rect)

class Game:
    def __init__(self):
        self.screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
        pygame.display.set_caption("벽돌깨기 게임 - 아이템 버전")
        self.clock = pygame.time.Clock()
        
        # 게임 객체 초기화
        self.balls = [Ball(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2)]
        self.paddle = Paddle(SCREEN_WIDTH // 2 - PADDLE_WIDTH // 2, SCREEN_HEIGHT - 50)
        self.bricks = []
        self.items = []
        self.score = 0
        self.lives = 50  # 목숨을 5개로 증가
        self.font = pygame.font.Font(None, 36)
        self.small_font = pygame.font.Font(None, 24)
        
        # 아이템 효과 타이머
        self.paddle_effect_timer = 0
        self.ball_effect_timer = 0
        self.power_ball_timer = 0
        
        self.create_bricks()
    
    def create_bricks(self):
        colors = [RED, ORANGE, YELLOW, GREEN, BLUE, PURPLE]
        for row in range(BRICK_ROWS):
            for col in range(BRICK_COLS):
                x = col * (BRICK_WIDTH + 5) + 35
                y = row * (BRICK_HEIGHT + 5) + 50
                color = colors[row % len(colors)]
                # 20% 확률로 아이템 벽돌 생성
                has_item = random.random() < 0.2
                self.bricks.append(Brick(x, y, color, has_item))
    
    def create_item(self, x, y):
        item_types = ['expand_paddle', 'shrink_paddle', 'multi_ball', 'extra_life', 
                     'slow_ball', 'fast_ball', 'power_ball']
        item_type = random.choice(item_types)
        return Item(x, y, item_type)
    
    def apply_item_effect(self, item_type):
        if item_type == 'expand_paddle':
            self.paddle.width = min(self.paddle.width * 1.5, 200)
            self.paddle_effect_timer = 600  # 10초
        
        elif item_type == 'shrink_paddle':
            self.paddle.width = max(self.paddle.width * 0.7, 50)
            self.paddle_effect_timer = 600
        
        elif item_type == 'multi_ball':
            # 현재 공들을 복사해서 추가
            new_balls = []
            for ball in self.balls:
                if len(self.balls) + len(new_balls) < 5:  # 최대 5개까지
                    new_ball = Ball(ball.x, ball.y)
                    new_ball.dx = ball.dx + random.randint(-2, 2)
                    new_ball.dy = ball.dy
                    new_balls.append(new_ball)
            self.balls.extend(new_balls)
        
        elif item_type == 'extra_life':
            self.lives += 1
        
        elif item_type == 'slow_ball':
            for ball in self.balls:
                ball.speed_multiplier = 0.5
            self.ball_effect_timer = 600
        
        elif item_type == 'fast_ball':
            for ball in self.balls:
                ball.speed_multiplier = 1.5
            self.ball_effect_timer = 600
        
        elif item_type == 'power_ball':
            self.power_ball_timer = 600
    
    def update_timers(self):
        # 패들 효과 타이머
        if self.paddle_effect_timer > 0:
            self.paddle_effect_timer -= 1
            if self.paddle_effect_timer == 0:
                self.paddle.width = self.paddle.original_width
        
        # 공 속도 효과 타이머
        if self.ball_effect_timer > 0:
            self.ball_effect_timer -= 1
            if self.ball_effect_timer == 0:
                for ball in self.balls:
                    ball.speed_multiplier = 1.0
        
        # 파워볼 타이머
        if self.power_ball_timer > 0:
            self.power_ball_timer -= 1
    
    def check_collisions(self):
        balls_to_remove = []
        
        for i, ball in enumerate(self.balls):
            # 공과 벽 충돌
            if ball.x <= ball.size or ball.x >= SCREEN_WIDTH - ball.size:
                ball.bounce_x()
            
            if ball.y <= ball.size:
                ball.bounce_y()
            
            # 공이 바닥에 떨어짐
            if ball.y >= SCREEN_HEIGHT:
                balls_to_remove.append(i)
                continue
            
            # 공과 패들 충돌
            if (ball.y + ball.size >= self.paddle.y and
                ball.x >= self.paddle.x and
                ball.x <= self.paddle.x + self.paddle.width and
                ball.dy > 0):
                
                # 패들의 어느 부분에 맞았는지에 따라 공의 방향 조정
                hit_pos = (ball.x - self.paddle.x) / self.paddle.width
                ball.dx = (hit_pos - 0.5) * 10
                ball.bounce_y()
            
            # 공과 벽돌 충돌
            ball_rect = pygame.Rect(ball.x - ball.size, ball.y - ball.size,
                                   ball.size * 2, ball.size * 2)
            
            for brick in self.bricks:
                if not brick.destroyed:
                    brick_rect = pygame.Rect(brick.x, brick.y, brick.width, brick.height)
                    if ball_rect.colliderect(brick_rect):
                        # 아이템 드롭
                        if brick.has_item:
                            item = self.create_item(brick.x + brick.width//2, brick.y + brick.height)
                            self.items.append(item)
                        
                        brick.destroyed = True
                        self.score += 10
                        
                        # 파워볼이 아닐 때만 튕김
                        if self.power_ball_timer <= 0:
                            ball.bounce_y()
                        break
        
        # 떨어진 공들 제거
        for i in reversed(balls_to_remove):
            if len(self.balls) > 1:
                self.balls.pop(i)
            else:
                # 마지막 공이 떨어졌을 때
                self.lives -= 1
                if self.lives > 0:
                    self.reset_balls()
    
    def update_items(self):
        items_to_remove = []
        
        for i, item in enumerate(self.items):
            item.move()
            
            # 화면 밖으로 나가면 제거
            if item.y > SCREEN_HEIGHT:
                items_to_remove.append(i)
                continue
            
            # 패들과 충돌 체크
            item_rect = pygame.Rect(item.x, item.y, item.width, item.height)
            paddle_rect = pygame.Rect(self.paddle.x, self.paddle.y, self.paddle.width, self.paddle.height)
            
            if item_rect.colliderect(paddle_rect):
                self.apply_item_effect(item.item_type)
                items_to_remove.append(i)
        
        # 아이템 제거
        for i in reversed(items_to_remove):
            self.items.pop(i)
    
    def reset_balls(self):
        self.balls = [Ball(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2)]
    
    def draw(self):
        self.screen.fill(BLACK)
        
        # 게임 객체 그리기
        for ball in self.balls:
            # 파워볼 효과
            if self.power_ball_timer > 0:
                pygame.draw.circle(self.screen, GOLD, (int(ball.x), int(ball.y)), ball.size + 3)
            ball.draw(self.screen)
        
        self.paddle.draw(self.screen)
        
        for brick in self.bricks:
            brick.draw(self.screen)
        
        for item in self.items:
            item.draw(self.screen)
        
        # UI 그리기
        score_text = self.font.render(f"점수: {self.score}", True, WHITE)
        lives_text = self.font.render(f"생명: {self.lives}", True, WHITE)
        balls_text = self.small_font.render(f"공: {len(self.balls)}", True, WHITE)
        
        self.screen.blit(score_text, (10, 10))
        self.screen.blit(lives_text, (10, 50))
        self.screen.blit(balls_text, (10, 90))
        
        # 활성 효과 표시
        effect_y = 120
        if self.paddle_effect_timer > 0:
            effect_text = self.small_font.render(f"패들 효과: {self.paddle_effect_timer//60 + 1}초", True, YELLOW)
            self.screen.blit(effect_text, (10, effect_y))
            effect_y += 25
        
        if self.ball_effect_timer > 0:
            effect_text = self.small_font.render(f"공 속도 효과: {self.ball_effect_timer//60 + 1}초", True, CYAN)
            self.screen.blit(effect_text, (10, effect_y))
            effect_y += 25
        
        if self.power_ball_timer > 0:
            effect_text = self.small_font.render(f"파워볼: {self.power_ball_timer//60 + 1}초", True, GOLD)
            self.screen.blit(effect_text, (10, effect_y))
        
        # 아이템 설명
        instructions = [
            "아이템 효과:",
            "초록(E): 패들 확장",
            "빨강(S): 패들 축소", 
            "파랑(M): 멀티볼",
            "분홍(+): 생명 추가",
            "하늘(SL): 공 감속",
            "주황(F): 공 가속",
            "보라(P): 파워볼"
        ]
        
        for i, instruction in enumerate(instructions):
            text = self.small_font.render(instruction, True, WHITE)
            self.screen.blit(text, (SCREEN_WIDTH - 150, 10 + i * 20))
        
        # 게임 오버 체크
        if self.lives <= 0:
            game_over_text = self.font.render("게임 오버! R키를 눌러 다시 시작", True, WHITE)
            text_rect = game_over_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2))
            self.screen.blit(game_over_text, text_rect)
        
        # 승리 체크
        elif all(brick.destroyed for brick in self.bricks):
            win_text = self.font.render("승리! R키를 눌러 다시 시작", True, WHITE)
            text_rect = win_text.get_rect(center=(SCREEN_WIDTH//2, SCREEN_HEIGHT//2))
            self.screen.blit(win_text, text_rect)
        
        pygame.display.flip()
    
    def reset_game(self):
        self.balls = [Ball(SCREEN_WIDTH // 2, SCREEN_HEIGHT // 2)]
        self.paddle = Paddle(SCREEN_WIDTH // 2 - PADDLE_WIDTH // 2, SCREEN_HEIGHT - 50)
        self.bricks = []
        self.items = []
        self.score = 0
        self.lives = 50
        self.paddle_effect_timer = 0
        self.ball_effect_timer = 0
        self.power_ball_timer = 0
        self.create_bricks()
    
    def run(self):
        running = True
        
        while running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                elif event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_r:
                        self.reset_game()
            
            # 키 입력 처리
            keys = pygame.key.get_pressed()
            if keys[pygame.K_LEFT] or keys[pygame.K_a]:
                self.paddle.move_left()
            if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
                self.paddle.move_right()
            
            # 게임이 진행 중일 때만 업데이트
            if self.lives > 0 and not all(brick.destroyed for brick in self.bricks):
                for ball in self.balls:
                    ball.move()
                self.check_collisions()
                self.update_items()
                self.update_timers()
            
            self.draw()
            self.clock.tick(60)
        
        pygame.quit()
        sys.exit()

# 게임 실행
if __name__ == "__main__":
    game = Game()
    game.run()

```
