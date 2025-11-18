
# 암산 테스트 코드

## 고령화 시대의 인지 훈련 솔루션

2025년 우리나라는 전체 인구 중 65세 이상 인구가 차지하는 비율이 20% 이상인 사회인 초고령 사회에 진입했다. 이에 따라 치매 예방과 인지 기능 유지에 대한 관심이 높아지고 있다. 복잡한 기기보다 친숙하고 직관적인 도구를 활용한 훈련이 필요하다고 생각한다. 
그래서 두 자리 수 암산 능력 향상 코드를 제작했다. 이 로봇은 간단한 터치 센서 조작 만으로 덧셈/뺄셈 문제를 풀게 하여 노인 층의 인지 기능 유지에 도움을 줄 수 있다.

---

##  1.  목표 및 하드웨어 구성

이 코드의 목표는 **다중 감각 피드백**을 통해 사용자의 집행 기능을 훈련하는 것이다. 사용자는 문제를 듣고(청각), 화면을 보고(시각), 손으로 센서를 눌러(운동/촉각) 암산 결과를 입력해야 한다.

| 부품                   | 역할            | 연결 포트       | 기능 상세                    |
| :------------------- | :------------ | :---------- | :----------------------- |
| **EV3 Brick**        | 메인 허브         | -           | 화면 표시, 음성 안내, 스피커 피드백    |
| **TouchSensor (S1)** | **10의 자리 입력** | **Port S1** | 한 번 누를 때마다 입력 값에 **+10** |
| **TouchSensor (S2)** | **1의 자리 입력**  | **Port S2** | 한 번 누를 때마다 입력 값에 **+1**  |
| **TouchSensor (S3)** | **확인/다음**     | **Port S3** | 입력 완료 및 다음 문제 진행         |

---

## 2. 파이썬 코드 전문

#!/usr/bin/env pybricks-micropython
from pybricks.hubs import EV3Brick
from pybricks.ev3devices import TouchSensor
from pybricks.parameters import Port, SoundFile, Color 
from pybricks.tools import wait, StopWatch 
import random


ev3 = EV3Brick()
touch_tens = TouchSensor(Port.S1)    # 10의 자리 입력
touch_ones = TouchSensor(Port.S2)    # 1의 자리 입력
touch_confirm = TouchSensor(Port.S3) # 확인/완료

#  상수
MIN_RESULT = 10 # 최소 결과값 (두 자리수)
MAX_RESULT = 99 # 최대 결과값

# 연산자 정의
OPERATORS = {
    '+': lambda a, b: a + b,
    '-': lambda a, b: a - b
}

# 핵심 함수 

def generate_math_problem():
    """결과가 10~99인 덧셈/뺄셈 문제 생성."""
    
    op_symbol = random.choice(list(OPERATORS.keys()))
    op_func = OPERATORS[op_symbol]

    while True:
        num1 = random.randint(30, 95)
        num2 = random.randint(10, 50)
        
        # 뺄셈이면 큰 수에서 작은 수를 빼도록 조정
        if op_symbol == '-':
            if num1 < num2:
                num1, num2 = num2, num1
                
        result = op_func(num1, num2)
        
        # 결과 범위 체크
        if MIN_RESULT <= result <= MAX_RESULT:
            return num1, op_symbol, num2, result


def announce_problem(num1, op_symbol, num2):
    """문제 화면 표시 및 음성 안내."""
    
    problem_text = "{} {} {} = ?".format(num1, op_symbol, num2)
    ev3.screen.clear()
    ev3.screen.print("Problem:")
    ev3.screen.print(problem_text)
    
    # 음성 안내 준비
    say_op = "plus" if op_symbol == '+' else "minus"
    say_text = "{} {} {} equals?".format(num1, say_op, num2)
    
    ev3.speaker.say(say_text)
    wait(500)


def get_user_input():
    # 입력 전, 센서 눌림 해제 대기
    while touch_tens.pressed() or touch_ones.pressed() or touch_confirm.pressed():
        wait(100)
    
    current_input = 0
    
    ev3.screen.clear()
    ev3.screen.print("Input: +10 (S1), +1 (S2)")
    ev3.screen.print("Press S3 to CONFIRM")
    
    # 입력 루프
    while True:
        ev3.screen.draw_text(20, 75, "Current: {}".format(current_input))
        
        # 10의 자리 입력 처리 (S1)
        if touch_tens.pressed():
            new_input = current_input + 10
            if new_input <= MAX_RESULT: 
                current_input = new_input
                ev3.speaker.beep(frequency=500, duration=50) 
            else:
                ev3.speaker.beep(frequency=300, duration=200) 
            wait(300) # 디바운싱
        
        # 1의 자리 입력 처리 (S2)
        if touch_ones.pressed():
            new_input = current_input + 1
            if new_input <= MAX_RESULT:
                current_input = new_input
                ev3.speaker.beep(frequency=700, duration=50) 
            else:
                ev3.speaker.beep(frequency=300, duration=200) 
            wait(300) # 디바운싱

        # 확인 (S3)
        if touch_confirm.pressed():
            ev3.speaker.play_file(SoundFile.GO) 
            wait(500)
            return current_input


def check_answer(target_result, user_input):
    """정답 확인 및 피드백."""
    
    if user_input == target_result:
        # 정답: 박수 소리, 녹색 라이트
        ev3.speaker.play_file(SoundFile.CHEERING)
        ev3.screen.print("Correct! Result: {}".format(target_result))
        ev3.light.on(Color.GREEN)
        wait(1000) 
        return True
    else:
        # 오답: 야유 소리, 빨간색 라이트
        ev3.speaker.play_file(SoundFile.BOO)
        ev3.screen.print("Wrong! Target: {}".format(target_result))
        ev3.screen.print("Your Input: {}".format(user_input))
        ev3.light.on(Color.RED)
        wait(1000) 
        return False


def main():
    """메인 게임 실행 및 점수 관리."""
    ev3.screen.clear()
    ev3.screen.print("--- MENTAL MATH START ---")
    ev3.speaker.play_file(SoundFile.START)
    wait(1000)
    
    score = 0
    
    while True:
        # 문제 생성/제시/입력
        num1, op_symbol, num2, target_result = generate_math_problem()
        announce_problem(num1, op_symbol, num2)
        user_input = get_user_input()
        
        # 채점
        is_correct = check_answer(target_result, user_input)
        ev3.light.off()
        
        if is_correct:
            score += 1
            ev3.screen.print("Score: {}".format(score))
            ev3.screen.print("Press S3 for next round.") 
        else:
            ev3.screen.print("Final Score: {}".format(score))
            ev3.screen.print("Press S3 to restart.") 
        
        # 다음 단계 대기
        while not touch_confirm.pressed():
            wait(100) 
        
        ev3.speaker.play_file(SoundFile.GO)
        wait(300)
        
        # 오답 시 점수 초기화
        if not is_correct:
            score = 0
            ev3.speaker.play_file(SoundFile.OKAY)
        
        wait(500)

main()