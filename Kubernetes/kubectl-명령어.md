|명령어|의미|
|--|--|
|kubectl version|클라이언트 버전은 kubectl의 버전이다. 서버의 버전은 master에 설치된 kubernetes의 버전이다.
|kubectl cluster-info|컨트롤 플레인의 가동정보, kubeDNS의 가동정보가 보인다. 클러스터의 구성 정보를 보여주는 듯 하다
|kubectl get nodes|우리의 애플리케이션을 호스팅할 수 있는 모든 노드를 보여준다. 노드의 이름, 상태, 역할, 나이, 버전 등을 볼 수 있다.
|kubectl get - |자원을 나열한다|
|kubectl describe - |자원에 대해 상세한 정보를 보여준다.
|kubectl logs - |파드 내 컨테이너의 로그들을 출력한다
|kubectl exec - |파드 내 컨테이너에 대한 명령을 실행한다.
|kubectl label - |새로운 라벨 할당
|kubectl scale -  |deployment를 이용한 스케일링
|kubectl set -|
|kubectl rollout undo deployments/...|이전 deployment로 롤백