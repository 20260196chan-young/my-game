# Week 11 실습

## 오늘 한 것
- PyInstaller 설치 및 빌드
- resource_path() 함수 추가
- --add-data 옵션으로 assets 포함
- .exe 실행 확인

## 빌드 명령어

```bash
pyinstaller --onefile --windowed --add-data "assets;assets" --name=polarity_shooter polarity_shooter.py
```

## resource_path()를 써야 하는 이유

exe로 빌드하면 기존 Python 실행 환경과 파일 경로가 달라져 이미지나 사운드 파일을 찾지 못하는 문제가 발생한다.  
resource_path() 함수를 사용하면 Python 실행 환경과 exe 실행 환경 모두에서 올바른 assets 경로를 찾을 수 있다.

## AI 활용 내역
- PyInstaller 설치 문제 해결
- exe 빌드 과정 도움
- assets 경로 문제 해결
- resource_path() 함수 적용 방법 설명
- --add-data 옵션 사용 방법 확인
