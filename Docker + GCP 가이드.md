

**1. Docker Image 서버 배포**

경로로 이동
```bash
cd C:\Users\mtaa5\workspace\Server\NewTrinity
```

서버 빌드

```bash
docker build -t trinity-server:latest .
```

서버 실행 

```bash
docker run -d --name trinity-server -p 8080:8080 trinity-server:latest
```
(서버 배포할 때는 8080 포트 사용해도 상관없음)


```bash
docker ps
```
실행 중인 서버 확인
![[Pasted image 20260402165513.png]]





[[**2.서버 활성화 후 GCP 연결**]]


docker image 파일을 GCP에 올리기

```bash
docker build -t asia-northeast3-docker.pkg.dev/helical-ion-460310-m0/test/trinity-server:v3 .
```

***주의 사항***

GCP에 올릴 때 먼저 진행해야 할 점 asia-northeast3에 접근하기 위해선 Artifact Registry에 권한을 만들어줘야합니다.


1. IAM에서 Assign roles에 **Artifact Registry Wirter** 추가하기
![[Pasted image 20260402170326.png]]

2. Aritifact Registry에 가서 **Location asia-northeast3**으로 하나 생성
![[Pasted image 20260402170435.png]]


완료 후 docker image 재 업로드
```bash
docker build -t asia-northeast3-docker.pkg.dev/helical-ion-460310-m0/test/trinity-server:v3 .
```

GCP에 인증
```bash
gcloud auth configure-docker asia-northeast3-docker.pkg.dev
```

docker 파일 Push

```bash
docker push asia-northeast3-docker.pkg.dev/helical-ion-460310-m0/test/trinity-server:v3
```

gcp에서 배포

```bash
gcloud run deploy trinity-server --image asia-northeast3-docker.pkg.dev/helical-ion-460310-m0/test/trinity-server:v3 --platform managed --region asia-northeast3
```

trinity-server:v3에서 v3은 버전을 나타냄 만약 버전을 올리게 되면 docker에 똑같은 새로운 버전에 이미지가 올라갑니다. 

![[Pasted image 20260402170737.png]]

배포 후 나오는 Service URL 구글에 작성

![[Pasted image 20260402170829.png]]

배포완료.


배포 완료 후 해야 할 점

ktor 설정에서 만들었졌던 application.conf에 

ktor {  
  deployment {  
    port = 8080  
    port = ${?PORT}  
    host = "0.0.0.0"  
  }  
  
  application {  
    modules = [ com.example.MainKt.module ]  
  }  
}

port 주소가 들어오면 그 주소로 진행하도록 변경
Cloud Run은 동적으로 포트를 할당하므로 반드시 `${PORT}`를 사용해야 한다

클라이언트 코드 변경

public async UniTask<UpgradeProductionResult> InitAsync()  
{  
    var request = UnityWebRequest.Get("https://trinity-server-413030808078.asia-northeast3.run.app/production");  
    request.SetRequestHeader("Authorization", _playerProfile.Uid);  
    await request.SendWebRequest();  
  
    if (request.result != UnityWebRequest.Result.Success)  
        throw new Exception(request.error);  
  
    var level = request.downloadHandler.text;  
  
    var productionLevelEntity = new ProductionLevel(int.Parse(level));  
  
    return new UpgradeProductionResult(new ProductionLevel(productionLevelEntity.Value), new GoldEntity(0),  
        new ProductionCost(productionLevelEntity.Value));  
}




만약 서버 코드가 변경되었다면?


재배포를 진행해야합니다.


*[[**2.서버 활성화 후 GCP 연결**]]*

서버 활성화 후 GCP 연결에 진행했던 배포 그대로 진행