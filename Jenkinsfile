// ---------------------------------------------------------------------------
// Jenkinsfile — CloudeDX 저장소에 둔다. reverdi(배포 저장소)가 아니다.
//
// 🔴 같은 저장소에 두면 무한 루프가 난다.
//    Jenkins 가 빌드 → values 에 태그 커밋 → 그 커밋이 Jenkins 를 다시 깨움 → 반복.
//    커밋하는 곳(reverdi)과 Jenkins 를 깨우는 곳(CloudeDX)이 달라야 한다.
//
// 파이프라인: lint → test → build → chart lint → update gitops
// 배포는 하지 않는다. Argo CD 가 git 을 보고 가져간다.
//
// 🔴 파드 정의를 이 파일 안에 둔다 (인라인).
//    helm-values/jenkins.yaml 의 podTemplates 에 의존하지 않는 이유:
//      · agent.enabled: false 면 차트가 podTemplates 를 렌더링하지 않는다
//      · Jenkins 를 다시 깔면 UI 설정이 사라진다
//      · 설정이 코드에 남아야 재현된다
// ---------------------------------------------------------------------------

// 빌드용 파드 — 컨테이너를 역할별로 나눈다.
// 한 이미지에 uv·buildah·helm·git 이 다 들어있지 않기 때문이다.
def BUILD_POD = '''
spec:
  # 🔴 IRSA — ECR push 권한
  #
  #    AWS 에서는 액세스 키를 넣지 않는다. 파드가 IAM 역할로 인증한다.
  #
  #    ⚠️ 이 SA 는 빌드 파드가 뜨는 네임스페이스(infra)에 있어야 한다.
  #       앱의 reverdi-batch 는 reverdi 네임스페이스라 여기서 못 쓴다.
  #       eksctl 로 따로 만든다 — aws/scripts/95-jenkins-irsa.sh 참조.
  #
  #    ⚠️ 로컬(Vagrant)에는 이 SA 가 없다. 그러면 파드가 안 뜬다.
  #       로컬에서 돌릴 때는 이 줄을 지우거나 SA 를 만들어야 한다:
  #         kubectl create sa jenkins-ecr -n infra
  serviceAccountName: jenkins-ecr

  # 🔴 배치 노드에서 빌드한다. 웹 노드의 자원을 쓰면 서비스 응답이 흔들린다.
  nodeSelector:
    workload: batch
  # 🔴 node4 에 taint(workload=batch:NoSchedule)가 걸려 있다.
  #    toleration 없이는 파드가 Pending 에서 멈춘다.
  tolerations:
    - key: workload
      operator: Equal
      value: batch
      effect: NoSchedule
  # ---------------------------------------------------------------------
  # 🔴 requests 는 "예약", limits 는 "상한" 이다 (2026-09-06 조정)
  #
  #   requests 합계가 노드의 할당 가능량보다 크면 파드가 영원히 Pending 이다.
  #     "0/6 nodes are available: 2 Insufficient cpu, 2 Insufficient memory"
  #
  #   로컬 node4 는 4 vCPU · 8GB 라 넉넉히 잡아뒀는데,
  #   AWS t3.medium 은 2 vCPU · 할당 가능 3.3GB 다. 그대로 쓰면 안 들어간다.
  #
  #   그래서 requests 만 낮추고 limits 는 그대로 둔다.
  #   → 스케줄링은 통과하고, 실제로 필요하면 limits 까지 쓴다.
  #
  #     requests 합계  2.0 vCPU / 4.2Gi  →  1.1 vCPU / 2.3Gi
  #     limits         변경 없음
  # ---------------------------------------------------------------------
  containers:
    # --- 파이썬 lint / test ---------------------------------------------
    - name: python
      image: python:3.13-slim
      command: ["sleep"]
      args: ["99d"]
      resources:
        # requests 를 낮춰 스케줄링을 통과시킨다. limits 는 그대로.
        #
        # ⚠️ 너무 낮추면 asyncpg 테스트가 이벤트 루프 오류로 실패한다.
        #    CPU 가 모자라면 커넥션 정리가 늦어져 다음 테스트가 그걸 물고 간다:
        #      "got Future ... attached to a different loop"
        #    GitHub Actions(2코어)에서는 안 나던 것이 300m 에서 났다.
        requests: { cpu: "800m", memory: "1Gi" }
        limits:   { cpu: "2",    memory: "2Gi" }

    # --- 테스트용 Postgres (사이드카) ------------------------------------
    # 🔴 docker run 으로 띄우지 않는다. k3s 는 containerd 라 Docker 데몬이 없다.
    #    같은 파드 안이라 127.0.0.1 로 접근된다.
    - name: postgres
      image: postgres:17-alpine
      env:
        - { name: POSTGRES_USER,     value: cloudedx }
        - { name: POSTGRES_PASSWORD, value: cloudedx }
        - { name: POSTGRES_DB,       value: cloudedx_test }
      resources:
        requests: { cpu: "200m", memory: "512Mi" }
        limits:   { cpu: "1",    memory: "1Gi" }

    # --- 이미지 빌드 -----------------------------------------------------
    - name: buildah
      image: quay.io/buildah/stable:latest
      command: ["sleep"]
      args: ["99d"]
      # 🔴 rootless 빌드에도 특별 권한이 필요하다.
      #    배치 전용 노드라 웹 파드에는 영향이 없다.
      securityContext:
        privileged: true
      resources:
        # 🔴 requests 를 절반으로. 빌드가 실제로 2Gi 를 상시 쓰지는 않는다.
        #    피크에는 limits(4Gi)까지 쓸 수 있다.
        requests: { cpu: "500m", memory: "1Gi" }
        # 크롤러 이미지가 2.7GB 라 상한은 여유를 준다
        limits:   { cpu: "2", memory: "4Gi" }

    # --- helm / git ------------------------------------------------------
    - name: tools
      # ⚠️ Alpine 기반이라 git·aws CLI 가 기본 포함되지 않는다.
      #    AWS 모드에서는 ECR 로그인 토큰을 받아야 해서 aws CLI 가 필요하다.
      #    아래 stage 에서 apk add 로 설치한다.
      #    ENTRYPOINT 가 helm 이므로 command 로 덮어써야 sh 가 돈다.
      image: alpine/helm:3.16.3
      command: ["sleep"]
      args: ["99d"]
      resources:
        requests: { cpu: "100m", memory: "256Mi" }
'''

pipeline {
    agent {
        kubernetes {
            yaml BUILD_POD
            defaultContainer 'python'
        }
    }

    parameters {
        // 기본값은 AWS. 로컬에서 돌리려면 빌드할 때 바꾼다.
        string(name: 'REGISTRY',
               defaultValue: '611669940814.dkr.ecr.ap-northeast-2.amazonaws.com',
               description: '이미지 레지스트리 (로컬: 192.168.56.15:30500)')
        choice(name: 'VALUES_FILE',
               choices: ['values-aws.yaml', 'values-vagrant.yaml'],
               description: '어느 환경의 값 파일에 태그를 커밋할지')
        // 🔴 임시 우회용. 기본은 비워둔다 — 테스트를 건너뛰는 건 예외 상황이다.
        //    예: app/tests/test_ownership.py
        string(name: 'DESELECT_TESTS', defaultValue: '',
               description: '건너뛸 테스트 경로 (비우면 전부 실행). 임시 우회용이며 상시로 쓰지 말 것')
    }

    options {
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
        // 🔴 문서 0-C IP 대역표와 일치해야 한다. AWS 에서는 ECR 주소로 교체.
        // 🔴 배포 대상에 따라 달라지는 값
        //
        //    로컬  REGISTRY=192.168.56.15:30500      · VALUES_FILE=values-vagrant.yaml
        //    AWS   REGISTRY=<계정>.dkr.ecr...        · VALUES_FILE=values-aws.yaml
        //
        //    Jenkins 잡 설정에서 파라미터로 넘기거나, 아래 기본값을 바꾼다.
        //    (Jenkins 관리 → 잡 → 이 빌드는 매개변수가 있습니다)
        REGISTRY    = "${params.REGISTRY}"
        // 🔴 배포 저장소. 앱 소스(CloudeDX)와 다른 곳이어야 한다.
        GITOPS_REPO = 'github.com/jpnjb0918-glitch/reverdi.git'
        CHART_PATH  = 'charts/reverdi'
        VALUES_FILE = "${params.VALUES_FILE}"
        AWS_REGION  = 'ap-northeast-2'
        DESELECT_TESTS = "${params.DESELECT_TESTS}"
    }

    stages {

        stage('준비') {
            steps {
                container('python') {
                    script {
                        // 🔴 GIT_COMMIT 은 체크아웃 이후에만 채워진다.
                        //    environment 블록에서 쓰면 null 이 되어 태그가 비어버린다.
                        env.IMAGE_TAG = env.GIT_COMMIT.take(7)
                        echo "이미지 태그: ${env.IMAGE_TAG}"
                    }
                    // python:3.13-slim 에는 uv 가 없다.
                    sh 'pip install --no-cache-dir uv && uv --version'
                }
            }
        }

        stage('lint') {
            steps {
                container('python') {
                    // ci.yml 과 동일하게 crawler extra 까지 포함해 검사한다.
                    sh '''
                        set -e
                        uv sync --extra crawler
                        uv run ruff check .
                    '''
                }
            }
        }

        stage('test') {
            steps {
                container('python') {
                    sh '''
                        set -e

                        # 사이드카가 접속을 받을 때까지 기다린다.
                        # 고정 sleep 은 느린 노드에서 부족하다.
                        for i in $(seq 1 30); do
                          if python -c "import socket;socket.create_connection(('127.0.0.1',5432),1)" 2>/dev/null; then
                            echo "DB 준비 완료"; break
                          fi
                          echo "DB 대기 중... ($i/30)"; sleep 2
                        done

                        # 🔴 crawler extra 를 설치하지 않는다.
                        #    백엔드가 실수로 크롤러를 최상단에서 임포트하면 여기서 걸린다.
                        uv sync

                        export DATABASE_URL="postgresql+asyncpg://cloudedx:cloudedx@127.0.0.1:5432/cloudedx_test"
                        export TEST_DATABASE_URL="$DATABASE_URL"

                        uv run alembic upgrade head
                        # 모델을 고치고 마이그레이션을 안 만든 경우가 여기서 걸린다.
                        uv run alembic check

                        # 🔴 DESELECT_TESTS 가 비어 있으면 전부 실행한다(기본).
                        #    값을 주면 그 경로만 건너뛴다 — 앱 쪽 문제를 조사하는 동안
                        #    파이프라인 나머지를 확인하려는 임시 우회다.
                        if [ -n "${DESELECT_TESTS}" ]; then
                          echo "⚠️  건너뛰는 테스트: ${DESELECT_TESTS}"
                          uv run pytest --deselect "${DESELECT_TESTS}"
                        else
                          uv run pytest
                        fi
                    '''
                }
            }
        }

        stage('build') {
            steps {
                container('buildah') {
                    // 🔴 k3s 는 containerd 라 docker build 가 안 된다. Buildah 를 쓴다.
                    //    Kaniko 는 2025년 6월 아카이브되어 쓰지 않는다.
                    // --tls-verify=false 는 사설 레지스트리가 HTTP 이기 때문
                    //    (infra/registries.yaml 의 insecure_skip_verify 와 짝)
                    // 🔴 ECR 은 인증이 필요하다. 로컬 레지스트리는 없었다.
                    //
                    //    IRSA 로 붙은 IAM 역할이 토큰을 발급받는다 — 액세스 키가 없다.
                    //    ECR 은 HTTPS 라 --tls-verify=false 도 필요 없다.
                    //
                    //    레지스트리 주소에 "amazonaws.com" 이 있으면 AWS 로 판단한다.
                    sh """
                        set -e

                        TLS_OPT="--tls-verify=false"

                        case "${REGISTRY}" in
                          *amazonaws.com*)
                            echo "ECR 로그인"
                            command -v aws >/dev/null 2>&1 || \
                              (command -v dnf >/dev/null && dnf install -y -q awscli) || \
                              (command -v apk >/dev/null && apk add --no-cache aws-cli) || \
                              pip install --no-cache-dir awscli
                            aws ecr get-login-password --region ${AWS_REGION} \
                              | buildah login --username AWS --password-stdin ${REGISTRY}
                            TLS_OPT=""
                            ;;
                        esac

                        buildah bud -f dockerfile.backend -t ${REGISTRY}/reverdi-backend:${IMAGE_TAG} .
                        buildah bud -f dockerfile.crawler -t ${REGISTRY}/reverdi-crawler:${IMAGE_TAG} .

                        buildah push \$TLS_OPT ${REGISTRY}/reverdi-backend:${IMAGE_TAG}
                        buildah push \$TLS_OPT ${REGISTRY}/reverdi-crawler:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('chart lint') {
            steps {
                container('tools') {
                    // 차트가 실제로 렌더링되는지 확인한다.
                    // 템플릿 오류를 클러스터에 올리기 전에 잡는다.
                    sh """
                        set -e
                        command -v git >/dev/null 2>&1 || apk add --no-cache git

                        rm -rf gitops-check
                        git clone --depth 1 https://${GITOPS_REPO} gitops-check

                        helm lint gitops-check/${CHART_PATH}
                        helm template reverdi gitops-check/${CHART_PATH} \\
                            -f gitops-check/${CHART_PATH}/${VALUES_FILE} > /dev/null
                        echo "차트 렌더링 정상"
                    """
                }
            }
        }

        stage('update gitops') {
            // main 브랜치에서만 커밋한다. PR 빌드가 배포를 일으키면 안 된다.
            when { branch 'main' }
            steps {
                container('tools') {
                    withCredentials([usernamePassword(
                            credentialsId: 'gitops-push-token',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN')]) {
                        // 🔴 이 단계만 배포 저장소에 커밋한다.
                        //    Jenkins 는 클러스터를 만지지 않는다. kubectl 도 쓰지 않는다.
                        //    작은따옴표라 Groovy 보간이 없어 토큰이 로그에 안 남는다.
                        sh '''
                            set -e
                            command -v git >/dev/null 2>&1 || apk add --no-cache git

                            rm -rf gitops-update
                            git clone https://${GIT_USER}:${GIT_TOKEN}@${GITOPS_REPO} gitops-update

                            cd gitops-update/${CHART_PATH}

                            # 🔴 sed 대신 python (2026-09-06 실패 → 수정)
                            #
                            #    sed -i "s|^  tag: .*|  tag: ...|" 가 동작하지 않았다.
                            #    Groovy 문자열을 지나며 따옴표가 벗겨져, 셸에서
                            #    공백으로 쪼개지고 "s|^" 만 스크립트가 됐다.
                            #
                            #    더 나빴던 건 종료 코드가 0 이었다는 점이다.
                            #    set -e 도 안 걸리고, git diff --quiet 이
                            #    "변경 없음"으로 판정해 조용히 넘어갔다.
                            #    로그에는 "태그 변경 없음"만 찍혔다.
                            #
                            #    python 은 정규식 치환 대신 문자열을 조립해
                            #    이스케이프 문제를 없앴고, 못 찾으면 명시적으로 실패한다.
                            python3 -c 'import pathlib,re,sys
f,t=sys.argv[1],sys.argv[2]
p=pathlib.Path(f); s=p.read_text(encoding="utf-8")
out=[]; n=0
for line in s.split("\n"):
    m=re.match(r"^(\s*)tag: ", line)
    if m:
        out.append(m.group(1)+"tag: \"%s\"" % t); n+=1
    else:
        out.append(line)
if n==0: sys.exit("tag 줄을 못 찾음: "+f)
p.write_text("\n".join(out), encoding="utf-8")
print("    %d곳 갱신 -> %s" % (n,t))' "${VALUES_FILE}" "${IMAGE_TAG}"

                            git config user.email 'jenkins@reverdi.local'
                            git config user.name  'jenkins-bot'

                            if git diff --quiet; then
                                echo "태그 변경 없음. 커밋을 건너뛴다."
                            else
                                git commit -am "ci: bump image tag to ${IMAGE_TAG}"
                                git push
                                echo "커밋 완료. Argo CD 가 감지해 배포한다."
                            fi
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            // 토큰이 들어간 디렉터리를 남기지 않는다.
            // 🔴 container() 로 감싸지 않는다 — 앞 단계가 실패하면 그 컨테이너가
            //    없을 수 있고, 그러면 정리 실패가 원래 오류를 가린다.
            sh 'rm -rf gitops-check gitops-update || true'
        }
        success { echo "빌드 성공 — 이미지 태그 ${env.IMAGE_TAG}" }
        failure { echo '빌드 실패. 위 로그에서 실패한 stage 를 확인할 것.' }
    }
}
// 배포는 여기서 하지 않는다 — Argo CD 의 selfHeal 이 위 커밋을 감지해 가져간다.
