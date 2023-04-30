# UnrealPractice: Troubleshooting  
『이득우의 언리얼 C++ 게임 개발의 정석』을 UE5로 실습했을 때 생긴 문제 해결 기록.  

## 목차  
* [Chapter 5. 폰의 제작과 조작](#Chapter-5.-폰의-제작과-조작)  
<hr/>  
  
# Chapter 5. 폰의 제작과 조작  
## AddMovementInput의 결과가 의도와 같지 않은 문제(p. 155)  
캐릭터가 함수 `AddMovementInput()`를 사용해 이동하도록 구현할 때 의도치 않은 문제가 발생한다.  
  
_1. 보는 방향에 따라 이동 속도가 다름_  
_2. 대각선으로 이동 시 속도가 빨라짐_  

### 원인  
우선 `AddMovementInput(Fvector Direction, float ScaleValue)`의 작동 원리를 살펴보자.  
Direction은 이동할 방향에 대한 벡터이고, ScaleValue는 입력에 적용해 해당 Vector에 곱해줄 scale 값이다.  
`AddMovementInput`은 내부적으로 $Direction * Value$(속도)를 계산하고 현재 위치 (0,0,0)에서 계산한 좌표로 이동시킨다.  
실습에서는 ScaleValue에 키보드 입력에 대한 값을 그대로 받아 처리한다.  

#### 1. 보는 방향에 따라 이동 속도가 다름  
`AddMovementInput()`이 계산한 속도가 가장 빨라지는 조건은 **이동 방향과 Direction의 방향이 같은 경우**이다.  

카메라가 어느 곳을 보고 있든, 이동에 대한 입력 방향은 따로 있으므로 이동하려는 방향의 Z값은 0이어야 한다.  
그런데 여기서 Direction은 카메라가 보는 방향을 반영하므로 **카메라가 보는 방향의 Z축 값을 포함**한다.  
따라서 _위나 아래를 보면서 캐릭터는 다른 방향으로 움직이고자 할 때_ Direction이 계산하는 방향 벡터의 힘이 줄어든다.  
  
#### 2. 대각선으로 이동 시 속도가 빨라짐  
캐릭터가 원점에 위치할 때, 실습에서 쓰인 함수가 각 입력의 상황에 호출되는 상황에 대해 기술하면 아래와 같다.
|상황|호출 함수|Direction|
|:---:|:---:|:---:|
|정면 이동|UpDown|(1,0,0)|
|우측 이동|LeftRight|(0,1,0)|
|정면 후측(대각선) 이동|UpDown + LeftRight|(1,1,0)|  
  
대각선 이동의 상황에서 Direction의 벡터 길이는 $\sqrt{2}$이 되며, 따라서 속도 또한 $\sqrt{2}$배가 된다.  

### 해결  
#### 1. 보는 방향에 따라 이동 속도가 다름  

Direction의 Z축 값을 무시하고 단위 벡터로 변경해 X, Y축의 값만 남긴다.
```C++
void AABPawn::UpDown(float NewAxisValue)
{
  FVector Direction = FRotationMatrix(GetControlRotation()).GetUnitAxis(EAxis::Y);
		Direction.Z = 0.0f;
		Direction.Normalize();
		AddMovementInput(Direction, NewAxisValue);
}
```

#### 2. 대각선으로 이동 시 속도가 빨라짐  
Direction을 단위 벡터로 변경하기 위해, 다음과 같은 과정을 거친다.  
  
1. 이동하려는 방향을 하나의 벡터로 변경한다.  
2. 방향 벡터를 단위 벡터로 변경한다.  
3. 매 Frame마다 `AddMovementInput`을 호출한다.  
  
1 ~ 2의 과정에서 Z축의 값을 없애고 DirectionG에 이동하려는 방향을 모두 합한다.  
3을 위해 `void Move()`를 추가해 매 Tick()마다 호출한다.  
이와 같은 방식으로 변경하면 속도가 매우 느려지기 때문에, `float VelocityG`를 추가해 scale값에 곱해준다.  
```C++
//ABPawn.h
UCLASS()
class ARENABATTLE_API AABPawn : public APawn{
...
protected:
	FVector DirectionG = FVector::Zero();
	float VelocityG = 100.0f;
...
private:
	void Move(float DeltaTime);
...
}

//ABPawn.cpp
...
void AABPawn::UpDown(float NewAxisValue)
{
		FVector Direction = FRotationMatrix(GetControlRotation()).GetUnitAxis(EAxis::X);
		Direction.Z = 0.0f;
		Direction.Normalize();
		DirectionG += Direction * FMath::Clamp(NewAxisValue, -1.0f, 1.0f);
}

void AABPawn::LeftRight(float NewAxisValue)
{
		FVector Direction = FRotationMatrix(GetControlRotation()).GetUnitAxis(EAxis::Y);
		Direction.Z = 0.0f;
		Direction.Normalize();
		DirectionG += Direction * FMath::Clamp(NewAxisValue, -1.0f, 1.0f);
}

void AABPawn::Move(float DeltaTime)
{
	if(DirectionG.isZero()){ return; }

	DirectionG.Normalize();
	AddMovementInput(DirectionG, VelocityG * DeltaTime);
	DirectionG.Set(0.0f, 0.0f, 0.0f);
}

void AABPawn::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
	Move(DeltaTime);
}
```

### 참고 자료  
* [UE5 문서: AddMovementInput](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/APawn/AddMovementInput/)  
* [AddMovementInput 함수의 이동속도 문제 해결하기](https://pppgod.tistory.com/39)  

<hr/>  
  
