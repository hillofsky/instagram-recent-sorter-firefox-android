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

## iPhone Safari에서 사용하기 (미검증)

> **iPhone Safari에서는 아직 실제 동작을 테스트하지 않았습니다.** 아래는 사용자 스크립트를 설치하는 방법이며, 설치 또는 Instagram Recent Sorter의 정상 작동을 보장하지 않습니다. 현재 공식 확인 대상은 Firefox Android입니다.

iPhone에서는 Instagram **앱이 아닌 Safari 브라우저**에서 시도할 수 있습니다. iOS 15.1 이상에서 동작하는 [Userscripts](https://github.com/quoid/userscripts) Safari 확장 프로그램을 사용합니다.

1. App Store에서 [Userscripts](https://apps.apple.com/app/id1463298887)를 설치합니다.
2. Userscripts 앱을 한 번 실행합니다. 스크립트 폴더가 자동 지정되지 않은 버전에서는 앱 안내에 따라 스크립트 폴더를 설정합니다.
3. iPhone **설정 → 앱 → Safari → 확장 프로그램 → Userscripts**에서 확장 프로그램을 켭니다. 사용하는 iOS 버전에 따라 메뉴 이름이 다를 수 있습니다. Safari에서 `instagram.com`에 대한 접근 권한도 허용합니다.
4. **Safari**에서 [사용자 스크립트 원본(.user.js)](https://raw.githubusercontent.com/hillofsky/instagram-recent-sorter-firefox-android/main/InstagramRecentMobile.user.js)을 엽니다.
5. Safari의 **페이지 메뉴 → 확장 프로그램 → Userscripts**를 열고 설치 안내가 나타나면 스크립트를 설치합니다. 설치 안내가 표시되지 않으면 `.user.js` 파일을 Userscripts 앱의 스크립트 폴더에 저장하는 방법도 있습니다.
6. Safari에서 [Instagram 웹사이트](https://www.instagram.com/)에 로그인하고 해시태그/검색 결과 페이지를 연 뒤 새로고침하여 **최신순 보기** 버튼이 나타나는지 확인합니다.

**알아둘 점:** 버튼 위치, 썸네일·영상 미리보기, 페이지 인식 및 사용자 스크립트 업데이트는 iPhone Safari에서 검증되지 않았습니다. **Instagram 공식 iPhone 앱에는 적용되지 않습니다.** iPhone에서 오류가 발생하면 실제 화면과 사용한 iOS·Safari·Userscripts 버전을 함께 제보해 주세요.

설치 방법 참고: [Userscripts 공식 문서](https://github.com/quoid/userscripts#ios-ipados) · [Apple Safari 확장 프로그램 안내](https://support.apple.com/guide/iphone/get-extensions-to-customize-safari-iphab0432bf6/ios)

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
