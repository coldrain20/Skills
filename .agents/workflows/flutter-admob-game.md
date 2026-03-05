---
description: Flutter + AdMob モバイルゲーム開発の手順とトラブル回避策
---

# Flutter + AdMob モバイルゲーム開発スキル

celery_game（女の子とセロリ）の開発経験から得たベストプラクティス集。
Flutter + Flame + AdMob のモバイルゲームを効率よく開発・公開するための手順。

---

## プロジェクト初期セットアップ

### 1. プロジェクト作成
```zsh
flutter create --org com.yourname app_name
cd app_name
```

### 2. 推奨ファイル構成
```
lib/
  main.dart              ← エントリーポイントのみ
  game/
    game.dart            ← ゲーム本体（FlameGame）
    player.dart          ← プレイヤー
    obstacle.dart        ← 障害物
  ui/
    title_screen.dart
    game_over_screen.dart
  services/
    ad_manager.dart      ← 広告管理（シングルトン）
    score_manager.dart   ← スコア保存（shared_preferences）
  config/
    constants.dart       ← 広告ID、ゲームパラメータ
```

### 3. 推奨パッケージ（pubspec.yaml）
```yaml
dependencies:
  flame: ^1.35.0
  google_mobile_ads: ^5.3.1
  shared_preferences: ^2.2.0

dev_dependencies:
  flutter_launcher_icons: ^0.13.1
  flutter_native_splash: ^2.4.4
```

---

## iOS ビルドエラー回避（最重要）

### Podfile テンプレート（必ず最初に設定）

// turbo-all

新規プロジェクト作成後、`ios/Podfile` を以下のように設定すること。
これを最初にやらないと、後で大量のビルドエラーに悩まされる。

```ruby
platform :ios, '14.0'

# CocoaPods の標準設定
ENV['COCOAPODS_DISABLE_STATS'] = 'true'

project 'Runner', {
  'Debug' => :debug,
  'Profile' => :release,
  'Release' => :release,
}

def flutter_root
  generated_xcode_build_settings_path = File.expand_path(File.join('..', 'Flutter', 'Generated.xcconfig'), __FILE__)
  unless File.exist?(generated_xcode_build_settings_path)
    raise "#{generated_xcode_build_settings_path} must exist."
  end
  File.foreach(generated_xcode_build_settings_path) do |line|
    matches = line.match(/FLUTTER_ROOT\=(.*)/)
    return matches[1].strip if matches
  end
  raise "FLUTTER_ROOT not found."
end

require File.expand_path(File.join('packages', 'flutter_tools', 'bin', 'podhelper'), flutter_root)
flutter_ios_podfile_setup

target 'Runner' do
  use_frameworks! :linkage => :static    # ← 静的リンク（重要）

  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))
  target 'RunnerTests' do
    inherit! :search_paths
  end
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '14.0'
      config.build_settings['ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES'] = 'YES'
      config.build_settings['ENABLE_BITCODE'] = 'NO'
      config.build_settings['ENABLE_USER_SCRIPT_SANDBOXING'] = 'NO'
    end

    # リソースバンドルのビルドエラー回避（Xcode 15/16対策）
    if target.respond_to?(:product_type) and target.product_type == "com.apple.product-type.bundle"
      target.build_configurations.each do |config|
        config.build_settings['CODE_SIGNING_ALLOWED'] = 'NO'
        config.build_settings['MACH_O_TYPE'] = 'staticlib'
        config.build_settings['EXECUTABLE_NAME'] = ''
      end
    end
  end
end
```

### 初回ビルド確認手順
```zsh
cd ios
pod install
cd ..
flutter build ios --simulator --debug --no-codesign
```
このコマンドが通ることを確認してから開発に入る。

---

## AdMob 統合テンプレート

### 広告IDの自動切り替え（Debug/Release）
```dart
import 'package:flutter/foundation.dart';

String get bannerAdUnitId {
  if (kDebugMode) {
    // テスト用ID
    return Platform.isAndroid
      ? 'ca-app-pub-3940256099942544/6300978111'
      : 'ca-app-pub-3940256099942544/2934735716';
  }
  // 本番用ID
  return Platform.isAndroid
    ? 'ca-app-pub-XXXX/XXXX'  // ← AdMobから取得したIDに差し替え
    : 'ca-app-pub-XXXX/XXXX';
}
```

### AndroidManifest.xml（android/app/src/main/AndroidManifest.xml）
```xml
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-XXXX~XXXX"/>
```

### iOS Info.plist（ios/Runner/Info.plist）
```xml
<key>GADApplicationIdentifier</key>
<string>ca-app-pub-XXXX~XXXX</string>
<key>SKAdNetworkItems</key>
<array>
  <dict>
    <key>SKAdNetworkIdentifier</key>
    <string>cstr6suwn9.skadnetwork</string>
  </dict>
</array>
```

---

## アイコン・スプラッシュ画面

### pubspec.yaml に追加
```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.1
  flutter_native_splash: ^2.4.4

flutter_launcher_icons:
  android: "launcher_icon"
  ios: true
  image_path: "assets/images/app_icon.png"
  min_sdk_android: 21

flutter_native_splash:
  color: "#テーマカラー"
  image: "assets/images/splash_logo.png"
  android: true
  ios: true
  fullscreen: true
```

### 生成コマンド
```zsh
dart run flutter_launcher_icons
dart run flutter_native_splash:create
```

---

## ストア申請の流れ

### Android（Google Play）
1. キーストア作成:
   ```zsh
   keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
   ```
2. `android/key.properties` にキーストア情報を記載
3. `build.gradle.kts` に署名設定を追加
4. `flutter build appbundle --release` でAAB生成
5. Google Play Console でアプリ登録 → AABアップロード → 審査提出

### iOS（App Store）
1. Apple Developer で Bundle ID 登録
2. Xcode → Signing & Capabilities → Team 設定
3. Xcode → Product → Archive
4. App Store Connect にアップロード → 審査提出

---

## 開発時の注意点

1. **Xcodeバージョン確認**: 開発開始前に `xcodebuild -version` と端末のiOSバージョンの整合性を確認
2. **git commit は細かく**: 動いたら即commit、ビルドが壊れたら戻せるように
3. **iOSの初回ビルドは最初にやる**: Podfileの設定とビルド確認を最優先
4. **google_mobile_ads のバージョン**: 最新版より1つ前の安定版を使う方が安全
5. **キーストアは絶対バックアップ**: 失うとアプリの更新ができなくなる
