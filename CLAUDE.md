# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 개요

이 저장소에는 브라우저 게임이 폴더마다 하나씩 들어 있습니다(`<폴더>/<파일>.html`). 루트의 `index.html`("태오 오락실")은 각 게임으로 연결되는 런처 페이지입니다. 게임 목록:

- `shooting/play.html`("뿅뿅 슈터"): 1~2인용 세로 스크롤 네온 슈팅 게임, 80 스테이지.
- `battle/play.html`("용사의 전설"): 3D 턴제 RPG. 5개 지역, 30 스테이지, 10스테이지마다 보스가 나오고, 대장간·뽑기·무기고와 2인 PvP 모드가 있습니다.
- `zombi/play.html`("좀비 아레나: 최후의 생존자"): 탑다운 웨이브 생존 슈팅 게임.

UI 문구와 코드 주석은 한국어입니다.

**게임 추가하기:** 폴더와 HTML 파일을 만든 뒤 `index.html`의 `GAMES` 배열에 항목(`path`, `title`, `icon`, `color`, `desc`, `tags`)을 추가합니다. 빌드 단계가 없어서 런처가 폴더를 자동으로 찾지 못하므로, `GAMES`에 없는 게임은 화면에 나오지 않습니다.

빌드 단계, 패키지 매니저, 린터, 테스트가 모두 없습니다. 각 게임은 하나의 독립된 HTML 파일이고 외부 의존성은 CDN에서만 불러옵니다. 게임마다 **three.js 버전이 다르므로** 게임끼리 코드를 그대로 복사하지 마세요.

- `shooting`: three.js r0.160.0을 ES 모듈로 사용(jsDelivr를 가리키는 `<script type="importmap">`), Google Fonts 사용.
- `battle`, `zombi`: three.js **r128**을 일반 `<script>` 태그로 불러와 전역 `THREE`로 사용. `battle`은 r128의 `examples/js` 후처리(postprocessing) 스크립트와 Sky 스크립트도 불러옵니다.

## 실행

`file://`에서는 ES 모듈 import가 동작하지 않으므로(`shooting`에 해당) HTTP로 띄웁니다.

```bash
python3 -m http.server 8000
```

그다음 `http://localhost:8000/`(런처)를 열거나 `http://localhost:8000/shooting/play.html`처럼 게임을 바로 엽니다. 변경 사항은 Playwright(Chromium 설치되어 있음)로 페이지를 열어 콘솔 오류가 없는지 확인합니다. 게임이 렌더링되려면 WebGL이 동작해야 합니다.

## `shooting/play.html` 구조

모든 코드가 하나의 `<script type="module">` 안에 다음 순서로 들어 있습니다.

1. **오디오**: 오디오 파일 없이 전부 WebAudio로 합성합니다. 효과음은 `laser`, `beep`, `noise`가 만듭니다. 스텝 시퀀서(`BGM`, `playStep`, `musicTick`)가 절차적 음악을 재생하고, `state`에 따라 `calm`, `play`, `boss` 트랙을 바꿉니다. `initAudio()`는 사용자 입력(제스처) 안에서 호출해야 합니다.
2. **게임 상태**: 객체가 아니라 모듈 수준 `let` 전역 변수입니다. `state`(`'title' | 'select' | 'play' | 'pause' | 'over' | 'allclear'`), `mode`(1인 또는 2인), 엔티티 배열 `players`, `pb`(플레이어 총알), `eb`(적 총알), `en`(보스 포함 적), `items`, `parts`, `rings`, `texts`. `reset()`이 이 값들을 모두 초기화합니다.
3. **스테이지 시스템**: `SECTORS` 8개에 각각 10 스테이지가 있습니다. 섹터마다 5스테이지는 엘리트, 10스테이지는 보스 스테이지입니다. `buildStage(n)`은 시드 고정 난수(`mulberry`)로 `addPattern`의 이름 붙은 패턴들을 조합해, 항상 같은 결과가 나오고 시간순으로 정렬된 스폰 이벤트 `script`를 만듭니다. 스테이지 안에서 `stagePhase`는 `intro → fight → clear` 순서로 진행됩니다. 난이도 `level`은 스테이지 번호로 정해집니다.
4. **좌표계**: 게임 로직은 고정된 450×800 논리 좌표(`W`, `H`)에서 돌아갑니다. 3D 씬에서는 `LX(x)`, `LZ(y)`(10으로 나눔)로 높이 `FLY`의 지면 평면에 매핑합니다. `toScreen`은 3D 좌표를 오버레이 좌표로 되돌리고, `toField`는 포인터 입력을 비행 평면에 레이캐스트합니다.
5. **렌더링은 캔버스 두 개를 겹쳐서** 합니다.
   - `#gl`: three.js 씬. `EffectComposer`와 `UnrealBloomPass`로 렌더링합니다. 엔티티의 Object3D는 필요할 때 만들어지고, `syncEnemies`, `syncItems`, `syncShips`가 로직 배열과 맞춰 주며 메시는 `e.obj`에 저장됩니다. 빛 파티클은 하나의 큰 `Points` 버퍼이고 매 프레임 `G()`/`GL()`로 채웁니다(최대 `MAXG`개). 레이저와 파편은 `InstancedMesh`를 씁니다. 재질은 색상별로 캐시하고(`edgeMat`, `basicMat`), 네온 느낌은 메시와 외곽선을 짝지어 주는 `addNeon`이 만듭니다.
   - `#c`: 2D 캔버스 오버레이. `drawUI()`와 `draw*` 헬퍼들이 HUD, 메뉴, 배너, 버튼을 그립니다. 같은 450×800 논리 좌표를 `dpr`로 스케일해서 씁니다.
6. **메인 루프**: `loop()`가 `update(dt)`, `render3D()`, `drawUI()`를 차례로 호출합니다. `dt`는 최대 0.033으로 제한됩니다. `hitStop`은 타격감을 위해 잠깐 시간을 느리게 합니다.

## 다른 게임 (요약)

- **`battle/play.html`**: 일반 `<script>` 하나. `MODE`는 `'pve'` 또는 `'pvp'`. 캐릭터는 기본 도형 헬퍼(`humanoid`, `applyRig`, `makeSlime`, `makeDragon` 등)로 만든 절차적 리그입니다. 메뉴는 HTML 패널(`hub`, `stages`, `forge`, `result`, `gacha`, `armory`, `pvp`)이고 `showPanel(id)`로 전환합니다. 진행 상황은 `SV` 객체에 있고 `save()`가 localStorage `hero_save2`에 저장합니다. 사운드 켜짐/꺼짐은 `snd`에 저장됩니다.
- **`zombi/play.html`**: 일반 `<script>` 하나. 2400×1800 아레나(`AW`, `AH`)에 사각형 장애물(`OBS`)이 있습니다. 데이터 테이블은 `WEAP`(무기), `ZT`(좀비 종류), `UPG`, `CON`입니다. 게임은 웨이브 단위로 진행되고(`startWave`, `spawnZombie`), 엔티티 배열은 `Z`, `B`, `EB`, `PT` 등입니다. 최고 웨이브는 `za_best`에 저장됩니다.

## 런처로 돌아가기

모든 게임에는 런처(`../index.html`)로 돌아가는 방법이 있어야 합니다. 새 게임도 마찬가지입니다.

- `shooting`: 타이틀 화면 왼쪽 위 ⌂ 버튼(`HOME_BTN`), 또는 타이틀 화면에서 Esc. 둘 다 `goHome()`을 거칩니다.
- `battle`: 메인 메뉴(`hub` 패널) 맨 아래의 "🏠 게임 목록" 버튼, 그리고 전투 중 상단 헤더의 같은 링크(사운드·메뉴 버튼 옆). 메뉴가 열리면 `body.menu`가 헤더를 숨기기 때문에 두 군데 모두 있어야 합니다.
- `zombi`: 시작 메뉴의 "🏠 게임 목록" 버튼("게임 시작" 옆).

## `shooting` 조작과 저장

- **조작**: 포인터 드래그(`drags` 맵으로 멀티터치 지원), 1P는 WASD, 방향키는 1인 모드에서 1P, 2인 모드에서 2P. P 또는 Esc로 일시정지, M은 음소거, B는 음악 켜기/끄기, Enter 또는 Space로 확인.
- **localStorage 키**: `ppyong_best`, `ppyong_best2`(모드별 최고 점수), `ppyong_stage1`, `ppyong_stage2`(모드별 해금된 최고 스테이지, 최대 81), `ppyong_music`. 모든 저장소 접근은 try/catch로 감싸져 있으니(세 게임 모두) 계속 그렇게 유지하세요.
