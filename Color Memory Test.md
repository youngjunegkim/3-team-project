
##  컬러 메모리 테스트 코드

이 코드는 심화된 기억력 및 인지 훈련 게임이다. EV3의 컬러 센서와 모터를 컨베이어 벨트에 연결하여 사용자에게 제시된 색깔 시퀀스를 로봇 스스로 규칙에 따라 찾아내고 보여준다.

목표: 사용자의 **작업 기억력**과 **규칙 적용 능력을 훈련하는 것에 중점을 두었다.


---

##  1. 하드웨어 구성 및 포트

이 코드는 컨베이어 벨트 위에 세 가지 색깔(빨강, 파랑, 초록) 블록이 순환하는 형태로 배치되어 있다고 가정한다.

| 부품 | 역할 | 연결 포트 | 기능 상세 |
| :--- | :--- | :--- | :--- |
| **EV3 Brick** | 메인 허브, 제어 및 피드백 | - | 화면, 음성, 라이트 출력 |
| **컨베이어 모터** | 블록 이동 제어 | **Port A** | 정밀 각도 이동을 통해 블록 정렬 |
| **컬러 센서** | 블록 색상 감지 | **Port S4** | 현재 위치의 색깔 감지 |
| **터치 센서 (R)** | 사용자 입력 (빨강) | **Port S1** | 빨강 선택 |
| **터치 센서 (B)** | 사용자 입력 (파랑) | **Port S2** | 파랑 선택 |
| **터치 센서 (G)** | 사용자 입력 (초록) | **Port S3** | 초록 선택 |

---

##  2. Python 코드 전문

#!/usr/bin/env pybricks-micropython
from pybricks.hubs import EV3Brick
from pybricks.ev3devices import Motor, ColorSensor, TouchSensor
from pybricks.parameters import Port, Stop, Direction, Color, SoundFile
from pybricks.tools import wait, StopWatch
import random

# --- 객체 설정 ---
ev3 = EV3Brick()
conveyor_motor = Motor(Port.A)
color_sensor = ColorSensor(Port.S4) 
touch_red = TouchSensor(Port.S1)    
touch_blue = TouchSensor(Port.S2)   
touch_green = TouchSensor(Port.S3)  

# --- 상수 ---
MOTOR_SPEED = 200 
MOVE_SPEED_SLOW = 140  
BLOCK_ANGLE = 270 # 블록 한 칸 이동 각도 (측정 필요)

COLOR_OPTIONS = [Color.RED, Color.BLUE, Color.GREEN]
COLOR_SOUND_MAP = {
    Color.RED: "RED", 
    Color.BLUE: "BLUE", 
    Color.GREEN: "GREEN"
}

# --- 핵심 함수 ---

def determine_move_angle(previous_color, current_color):
    """규칙에 따라 이동할 모터 각도 (양수: CW, 음수: CCW) 결정."""
    
    if previous_color == current_color:
        return 0

    if previous_color == Color.RED:
        return BLOCK_ANGLE # R 다음은 무조건 CW
    
    elif previous_color == Color.GREEN:
        if current_color == Color.BLUE or current_color == Color.RED:
            return -BLOCK_ANGLE # G 다음은 무조건 CCW
        
    elif previous_color == Color.BLUE:
        if current_color == Color.GREEN:
            return BLOCK_ANGLE # B 다음 G: CW
        elif current_color == Color.RED:
            return -BLOCK_ANGLE # B 다음 R: CCW
            
    if previous_color is None:
        if current_color == Color.GREEN:
            return -BLOCK_ANGLE
        elif current_color == Color.RED:
            return BLOCK_ANGLE
        
    return 0 


def move_to_target_color(target_color, previous_color):
    """규칙 각도만큼 정밀하게 모터를 움직여 블록 앞으로 이동."""
    
    angle_to_move = determine_move_angle(previous_color, target_color)
    
    if angle_to_move == 0:
        ev3.screen.print("Same Color or Error")
        wait(1000)
        return
        
    conveyor_motor.run_angle(
        speed=MOVE_SPEED_SLOW, 
        rotation_angle=angle_to_move, 
        then=Stop.HOLD, 
        wait=True
    )
    
    ev3.screen.print("Moved to Target.")
    wait(500)
    
def generate_new_sequence(current_sequence):
    """현재 시퀀스에 새 색깔 추가 (연속 중복 방지)."""
    
    last_color = current_sequence[-1] if current_sequence else None
    available_options = list(COLOR_OPTIONS)
    
    if last_color in available_options:
        available_options.remove(last_color)
        
    if available_options:
        new_color = random.choice(available_options)
    else:
        new_color = random.choice(COLOR_OPTIONS)

    current_sequence.append(new_color)
    return current_sequence

def present_sequence(sequence):
    """로봇이 시퀀스를 제시하고 사용자 입력을 기다림."""
    ev3.screen.clear()
    ev3.screen.print("--- Presenting ---")
    
    # 모터 각도 0으로 초기화
    conveyor_motor.run_angle(speed=MOTOR_SPEED, rotation_angle=-conveyor_motor.angle(), then=Stop.HOLD, wait=True)
    
    previous_color = Color.BLUE # 시작 위치 BLUE 가정
    
    for color in sequence:
        
        move_to_target_color(color, previous_color) 
        
        color_name = COLOR_SOUND_MAP.get(color, "UNKNOWN")
        ev3.speaker.say(color_name)
        
        ev3.screen.print("Current: {}".format(str(color)))
        ev3.light.on(color) 
        wait(1000) 
        ev3.light.off()
        
        previous_color = color
        
    ev3.speaker.beep(frequency=1000, duration=300) 
    ev3.screen.print("--- Your Turn! ---")
    ev3.screen.print("Press Sensor (1=R, 2=B, 3=G)") 


def get_user_input():
    """터치 센서 입력 받고 색깔 반환."""
    while True:
        if touch_red.pressed():
            return Color.RED
        if touch_blue.pressed():
            return Color.BLUE
        if touch_green.pressed(): 
            return Color.GREEN
        wait(10)


def check_sequence(sequence):
    """사용자 입력과 시퀀스 비교."""
    for expected_color in sequence:
        ev3.screen.print("Waiting for: {}".format(str(expected_color)))
        actual_color = get_user_input()
        wait(200) 
        
        if actual_color != expected_color:
            return False 
        
        ev3.speaker.beep(frequency=1500, duration=100)
        
    return True

# --- 메인 게임 루프 ---

def main():
    ev3.screen.clear()
    ev3.screen.print("--- Memory Game START ---")
    ev3.speaker.play_file(SoundFile.START)
    
    current_sequence = []
    round_number = 0
    
    conveyor_motor.reset_angle(0)
    
    while True:
        round_number += 1
        ev3.screen.clear()
        ev3.screen.print("--- Round {} ---".format(round_number))
        wait(1000)
        
        current_sequence = generate_new_sequence(current_sequence)
        
        present_sequence(current_sequence)
        
        ev3.screen.print("Checking input...")
        is_correct = check_sequence(current_sequence)
        
        if is_correct:
            ev3.speaker.play_file(SoundFile.FANFARE)
            ev3.screen.print("SUCCESS! Level {} passed.".format(round_number))
            wait(2000)
        else:
            ev3.speaker.play_file(SoundFile.BOO)
            ev3.screen.print("GAME OVER! Score: {}".format(round_number - 1))
            
            ev3.screen.print("Returning to start (BLUE)")
            move_to_target_color(Color.BLUE, current_sequence[-1] if current_sequence else None) 
            break

main()