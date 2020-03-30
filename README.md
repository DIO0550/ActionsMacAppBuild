# acitonsMacAppBuild

## 概要
tagプッシュ時にビルドを実行し、Relaseのassetsにビルド物を追加するActions。

## 説明
### - タグ名取得
```yml
 - uses: olegtarasov/get-tag@v1
   id: tagName
 - name: GET TAG NAME 
   run: |
         echo "::set-env name=TAG_NAME::${{ steps.tagName.outputs.tag  }}"
```
プッシュされたタグ名を環境変数「TAG_NAME」に設定

### - ビルド
1. xcodebuildを使用できるようにする。
```yml
- uses: actions/checkout@v1
- name: xcode-select 
  run: sudo xcode-select --switch /Applications/Xcode_11.3.app
```

2. achirveビルドを実行する。
```yml
- name: CREATE DIR
   run: |
        mkdir archive

- name: BUILD APP
   run: |
        xcodebuild -project './ActionsMacAppBuild/ActionsMacAppBuild.xcodeproj' -config Develop -scheme 'ActionsMacAppBuild' -archivePath ./archive/ActionsMacAppBuild archive
```
configで指定するものは、署名なしのものを指定する。

3. appファイルをzip化
```yml
- name: ZIP APP
   run: |
        zip ActionsMacAppBuild -r ./archive/ActionsMacAppBuild.xcarchive/Products/Applications/ActionsMacAppBuild.app
```
xcodebuildのexportでは、必ず署名が走るため、アーカイブファイル内のappファイルをzip化する。


### - アップロード
1. releaseの作成
```yml
- name: CREATE RELEASE
   run: |
        curl -H 'Content-Type:application/json; charset=UTF-8' -H 'X-Accept:application/json' -X POST -d '{ "tag_name" : "${{ env.TAG_NAME }}" }' https://api.github.com/repos/${{ github.repository }}/releases?access_token=${{ secrets.GITHUB_TOKEN }}
```
タグのプッシュのみだと、releaseが作成されないので、タグ名からreleaseを作成する。

2. release_idの取得
```yml
- name: RELEASE ID
   run: |
        echo "::set-env name=RELEASE_ID::$(curl https://api.github.com/repos/${{ github.repository }}/releases/tags/${{ env.TAG_NAME }}?access_token=${{ secrets.GITHUB_TOKEN }} | jq '."id"')"
```
1で作成したreleaseのidを取得する。(release_id ≠ タグ名) 

3. release_idからassetsをアップロードする。

```yml
- name: UPLOAD APP
   run: |
        curl -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
             -H "Accept: application/vnd.github.manifold-preview" \
             -H "Content-Type: application/zip" \
             --data-binary @ActionsMacAppBuild.zip \
             "https://uploads.github.com/repos/${{ github.repository }}/releases/${{ env.RELEASE_ID }}/assets?name=ActionsMacAppBuild.zip"
```

## MacApp以外での使用
ビルド処理を変更することで、MacAppのビルド以外でも使用可能。  
