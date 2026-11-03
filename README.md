# Instagram Recent Sorter

Firefox Android에서 Instagram 검색 결과를 최신순으로 정렬해 볼 수 있는 Tampermonkey 사용자 스크립트입니다.

> 비공식 도구이며 Instagram 또는 Meta와 제휴하거나 승인받지 않았습니다.

## 주요 기능

- 해시태그 및 검색 결과 화면에서 **최신순 보기** 버튼 제공
- 현재 화면에서 확인할 수 있는 게시물을 최신순으로 정렬하여 별도 목록에 표시
- 게시물 썸네일과 추정 게시 시각 표시
- 썸네일을 1~6열로 조정 (기본 3열)
- Instagram 원본 게시물은 그대로 두고 별도 화면에서 탐색

## 설치 (Firefox Android)

1. Firefox Android에 **Tampermonkey** 확장 기능을 설치합니다.
2. 아래 **스크립트 설치** 링크를 Firefox로 엽니다.
3. Tampermonkey에서 설치를 확인합니다.
4. Instagram 검색 결과를 열어 페이지를 새로고침하고 **최신순 보기**를 누릅니다.

**[스크립트 설치 (Tampermonkey)](https://raw.githubusercontent.com/hillofsky/instagram-recent-sorter-firefox-android/main/InstagramRecentMobile.user.js)**

설치 화면이 열리지 않으면 [`InstagramRecentMobile_AndroidDownload.txt`](InstagramRecentMobile_AndroidDownload.txt)의 내용을 복사해 Tampermonkey 새 스크립트 편집기에 붙여넣을 수 있습니다. 이미 같은 스크립트를 설치했다면 중복 실행되지 않도록 기존 것을 비활성화하세요.

## 사용 방법

검색 결과 화면에서 **최신순 보기**를 누르면 별도 목록이 열립니다. 목록의 열 수를 변경하거나 다시 스캔할 수 있습니다. **게시물 더 불러오기**는 사용자가 직접 선택했을 때만 원본 화면을 한 번 스크롤합니다.

## 알아둘 점

- **게시 시각은 추정값**이며 Instagram이 제공하는 공식 게시 시간이 아닙니다.
- Instagram이 화면에 표시한 게시물만 정렬합니다. 검색 결과 전체를 가져오거나 누락된 게시물을 자동으로 찾지는 않습니다.
- 썸네일·영상 미리보기 로딩 과정에서 브라우저가 미디어를 추가로 요청할 수 있습니다.
- Instagram의 화면 구조 또는 정책이 바뀌면 작동하지 않을 수 있으며, 사용에 따른 계정 제한 위험이 없음을 보장하지 않습니다.

## 버전

**v0.1** — 최초 공개 버전

## 라이선스

현재 별도의 오픈소스 라이선스가 지정되어 있지 않습니다. 저장소 공개가 제3자에게 수정·재배포 권한을 자동으로 부여하지는 않습니다.
