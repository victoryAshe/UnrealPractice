# UnrealPractice: Troubleshooting  
『이득우의 언리얼 C++ 게임 개발의 정석』을 UE5로 실습했을 때 생긴 문제 해결 기록.  

## 목차  
* [Chapter 5. 폰의 제작과 조작](#chapter-5-폰의-제작과-조작)  
* [Chapter 6. 캐릭터의 제작과 컨트롤](#chapter-6-캐릭터의-제작과-컨트롤)  
* [Chapter 7. 애니메이션 시스템의 설계](#chapter-7-애니메이션-시스템의-설계)
* [Chapter 8. 애니메이션 시스템 활용](#chapter-8-애니메이션-시스템-활용)
* [Chapter 9. 충돌 설정과 데미지 전달](#chapter-9-충돌-설정과-데미지-전달)
* [Chapter 10. 아이템 상자와 무기 제작](#chapter-10-아이템-상자와-무기-제작)
<hr/>  
  
# Chapter 5. 폰의 제작과 조작  
## AddMovementInput의 결과가 의도와 같지 않은 문제(p. 155)  
캐릭터가 함수 `AddMovementInput()`를 사용해 이동하도록 구현할 때 의도치 않은 문제가 발생한다.  
  
_1. 보는 방향에 따라 이동 속도가 다름_  
_2. 대각선으로 이동 시 속도가 빨라짐_  
  
실제 게임에서 이런 일이 발생하면 게임 플레이에 꽤나 영향을 미치므로 해결 방법을 찾아보았다.  

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
### 이후 챕터 실습 에 대한 변경 사항  
Chapter 6 실습에서 DIABLO 방식과 GTA 방식에 대해 따로 코딩을 해주는데,  
GTA 방식이 기존에 실습했던 방식이므로 `switch case`문에서 GTA의 부분만 위와 같이 진행하면 된다. 
+ TODO: DIABLO 방식에서도 적용하기
    
### 참고 자료  
* [UE5 문서: AddMovementInput](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/APawn/AddMovementInput/)  
* [AddMovementInput 함수의 이동속도 문제 해결하기](https://pppgod.tistory.com/39)  

<hr/>  
  
# Chapter 6. 캐릭터의 제작과 컨트롤  
## SpringArm->RelativeRotation(p.203 - p. 204)  
책과 똑같이 코딩하면 일부 코드가 에러를 발생시킨다.
```C++
// 에러 발생 부분
void AABCharacter::Tick(float DeltaTime)
{
	...
	SpringArm->RelativeRotation = FMath::RInterpTo(SpringArm->RelativeRotation, ArmRotationTo, DeltaTime, ArmRotationSpeed);
	break;
	...
}
```
### 원인  
SpringArm의 RelativeRotation이 private으로 바뀌었기 때문에 생긴 문제였다.  
  
### 해결  
```C++
void AABCharacter::Tick(float DeltaTime)
{
	...
	SpringArm->SetRelativeRotation(FMath::RInterpTo(SpringArm->GetRelativeRotation(), ArmRotationTo, DeltaTime, ArmRotationSpeed));
	break;
	...
}
```
  
<hr/>  
  
# Chapter 7. 애니메이션 시스템의 설계  
## Animation Retargeting 방법이 다름 (p.233 - p.238)  
UE5는 이전 버전과 다른 Animation Retargeting 방식을 사용한다.  
  
### 해결  
참고 자료와 같은 방식을 사용하되, Source Skeletal Mesh는Characters\Mannequins\Meshes\SKM_Manny를 이용했다.  
  
### 참고 자료  
* [[UE5] Animation Retargeting 방법](https://devjino.tistory.com/276)  
  
## JumpEnd Animation에서 캐릭터가 사라짐 (p.241)  
Retarget할 Animation들은 Characters\Mannequins\Animations\Manny\에서 가져왔으며,  
각각 JumpStart: MM_Jump, JumpLoop: MM_Fall_Loop, JumpEnd: MM_Land를 사용했다.  
  
Animation 창에서 봤을 때는 모두 잘 재생되었는데,  
Anim Graph에서 연결 뒤 재생하면 JumpEnd에서 캐릭터가 작아지면서 사라졌다가 Idle에서는 다시 나타나는 문제가 발생했다.  
  
### 원인  
MM_Land에서 Retargeting한 Animation에서 Additive Setting>AdditiveAnimType이 Local Space로 설정되어있어 나타난 문제였다.  
 
###  해결  
해당 Animation 창에서 Additive Setting>AdditiveAnimType을 No Additive로 설정하니 정상 작동하는 것을 확인했다.  
  
### 참고 자료  
* [Epic Games Dev Community Forum](https://forums.unrealengine.com/t/player-character-disappears-when-jumping/478345)  
  
<hr/>  
  
# Chapter 8. 애니메이션 시스템 활용
## Category = Attack이 Editor에서 표시가 안 되는 문제

### 원인
솔루션 빌드를 해도 Editor에 반영이 안 되어서였음;

### 해결
Editor에서 즉석 컴파일 및 로드 버튼을 클릭

## 콤보 공격의 구현 문제
Anim notify를 어떻게 배치해보아도 Combo가 1->2로 넘어가다가 말았다.  
Editor상에서 확인해보니 Combo는 1->2까지 측정이 되는데 게임 플레이 상에서 Montage Section이 다음으로 넘어가지 않는 것 같았다.  
콘솔 로그를 봐도 Error가 찍히지 않아 난감했다.  

### 원인 1: BlendOut Trigger Time
Anim Montage의 BlendOut Trigger Time은 해당 Section이 시작된 후 얼마나 지나야 Blend를 시작할지 결정한다.  
이 기본값이 Section1의 길이를 넘어갔기 때문에 NextAttackCheck Notify가 제대로 체크될 수 없었다.  

### 해결 1
BlendOut Trigger Time을 0으로 설정한다.  
그러면 NextAttackCheck를 Section의 너무 끝에 두지 않아도 잘 작동한다.  

### 원인 2: 코딩 실수😨
![Untitled](https://github.com/victoryAshe/UnrealPractice/assets/80302657/dd514246-335e-4f54-a32c-2d4495069650)
`GetAttackMontageSectionName(int32 Section)`은 Section 이름을 만들어 넘겨주는 것이었는데,  
그만 로그로 착각하는 바람에... `TEXT("Attack %d")`로 쓸데없는 띄어쓰기를 해서 Section 이름이 제대로 넘어가지 않았다.  

### 해결 2
잘못 쓴 부분을 제대로 고쳐주었더니 콤보가 말끔하게 작동되었다.

### 참고 자료
* [Anim Montage의 BlendOut Trigger Time 설정](https://blog.naver.com/PostView.naver?blogId=kzh8055&logNo=222044403660)

<hr/>

# Chapter 9. 충돌 설정과 데미지 전달
## HitResult->GetActor (p. 298, 305)
HitResult에서 Actor의 Valid 체크를 하는 부분에서 계속 에러가 났다.
### 원인
책의 버전에서와 달리 HitResult가 Actor를 private처리하는 바람에 생긴 문제였다.
UE5에서는 `HitResult.Actor.IsValid()` 대신 `HitResult.GetActor()`로 Actor의 유효성을 검증한다.
GetActor()가 실패하면 null이 반환되기 때문.
### 해결
```C++
//p. 298
if(HitResult.Actor.IsValid())
{
	ABLOG(Warning, TEXT("Hit Actor Name : %s"), *HitResult.Actor->GetName());
}

//->UE5
if(HitResult.GetActor())
{
	ABLOG(Warning, TEXT("Hit Actor Name : %s"), *HitResult.GetActor()->GetName());
}

//p.305
if(HitResult.Actor.IsValid())
{
	...
			FDamageEvent DamageEvent;
			HitResult.Actor->TakeDamage(50.0f, DamageEvent, GetController(), this);
}

//->UE5
if (bResult)
	{
		if (HitResult.GetActor())
		{
...
			FDamageEvent DamageEvent;
			HitResult.GetActor()->TakeDamage(50.0f, DamageEvent, GetController(), this);
		}
	}
```

### 참고자료
* [언리얼 문서](https://docs.unrealengine.com/5.2/en-US/API/Runtime/Engine/Engine/FHitResult/)

<hr/>

# Chapter 10. 아이템 상자와 무기 제작
## Box의 Build Scale 변경 후 Static Mesh 깜빡임 문제(p. 326)
![boxflickering](https://github.com/victoryAshe/UnrealPractice/assets/80302657/09618b55-db4e-4f16-8914-8c37cc955b63)

### 원인  
해당 box의 viewport에서 Show - visualize - OutOfBoundPixels 를 켜주면 바운드를 벗어난 mesh의 부분이 노란색으로 표시된다.  
z값 bound는 buildscale로 인해 늘어난 mesh보다 낮은 곳에 위치해 있기 때문에 bound culling이 계속해서 일어나 깜빡거리는 것으로 보인다.  
  
Bounds Scale 은 액터의 바운드를 균등 스케일 조절하여, 원래 스케일 값의 배수 역할을 한다.  
그 외에 비균등 스케일은 Positive Bounds Extension (양수 바운드 확장) 및 Negative Bounds Extensions (음수 바운드 확장)으로 조절하는데,  
최종 바운드는 [Imported Bound - Negative Bound]에서 [Imported Bound + Positive Bound]까지이다.  

### 해결 
![화면 캡처 2023-06-18 065407](https://github.com/victoryAshe/UnrealPractice/assets/80302657/03ef51ad-f928-45ff-b6d4-bbc279fade8f)  
  
positive bound를 잘 확인하면서 z값을 조금씩 확장해준다.  
bound를 너무 늘리면 culling 가능성을 낮춰 성능에 영향을 끼칠 수 있으므로, 문제를 일으키지 않는 선까지만 늘려주는 것이 좋다.  
  
### 참고 자료
* [언리얼 포럼](https://forums.unrealengine.com/t/flickering-mesh/312889)  
* [언리얼 문서 1](https://docs.unrealengine.com/4.27/ko/RenderingAndGraphics/VisibilityCulling/)  
* [언리얼 문서 2](https://docs.unrealengine.com/4.26/en-US/API/Runtime/Engine/Engine/USkeletalMesh/PositiveBoundsExtension/)  
* [Unreal Skeletal mesh bound culling](https://mentum.tistory.com/204)
  
## VisualStudio에서 _Delegate_.AddDynamic이 없다고 뜨거나 에러 표시되는 문제(p. 333)  
### 원인  
VisualSudio의 Intellisense가 AddDynamic을 찾지 못해서 생기는 문제이다.🙃  
그냥 무시하고 진행하다보면 에러 표시가 사라져있을 것.  

### 참고 자료  
* [언리얼 델리게이트](https://velog.io/@myverytinybrain/%EC%96%B8%EB%A6%AC%EC%96%BC-%EB%8D%B8%EB%A6%AC%EA%B2%8C%EC%9D%B4%ED%8A%B8)

<hr/>
