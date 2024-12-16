---
title: "[Flutter] SharedPreferencesからSharedPreferencesAsync/WithCacheへの移行"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Flutter]
published: true
---

# 概要

[shared_preferences](https://pub.dev/packages/shared_preferences)のバージョン 2.3.0 以降では、新たに`SharedPreferencesAsync/WithCache`が提供されています。
特に Android では内部実装が変更され、従来の[Android Shared Preferences](https://developer.android.com/reference/android/content/SharedPreferences)から[DataStore Preferences](https://developer.android.com/topic/libraries/architecture/datastore)に切り替わりました。
しかし、[Android Shared Preferences](https://developer.android.com/reference/android/content/SharedPreferences)と[DataStore Preferences](https://developer.android.com/topic/libraries/architecture/datastore)ではデータの保存場所が異なるため、`SharedPreferencesAsync/WithCache`へ移行するためには、手動でデータを移行する必要があります。
ここでは、その移行のためのサンプルコードを紹介します。

# コード

以下は、`SharedPreferences` から `SharedPreferencesAsync` に移行するためのコード例です。

```dart

/// 最新のPreferenceのバージョン
const _latestPreferenceVersion = 2;

Future<void> migratePreferenceIfNeeded() async {
  final prefs = SharedPreferencesAsync();
  final currentPreferenceVersion = await prefs.getInt(PreferenceKey.currentPreferenceVersion.name) ?? 1;

  // すでに最新のバージョンのPreferenceを使用している場合はマイグレーション不要
  if (currentPreferenceVersion == _latestPreferenceVersion) {
    logger.info('No need to migrate preference.');
    return;
  }

  // バージョン1からバージョン2へのマイグレーション
  if (currentPreferenceVersion < 2) {
    await _migrateToV2();
  }

  // バージョンを最新に更新
  await prefs.setInt(PreferenceKey.currentPreferenceVersion.name, _latestPreferenceVersion);
  logger.info('Migrated preference to version $_latestPreferenceVersion');
}

/// バージョン2へのマイグレーション
///
/// 内容
/// - [SharedPreferences]から[SharedPreferencesAsync], [SharedPreferencesWithCache]への変更
///   - Androidの場合は内部的にSharedPreferenceからDataStoreへの変更が行われているため、手動でデータをコピーしている
///   - iOSの場合は、UserDefaultsのままなのでマイグレーションの作業必要なし
Future<void> _migrateToV2() async {
  logger.info('_migrateV2');
  if (Platform.isAndroid) {
    logger.info('Android detected: Migrate from SharedPreferences to SharedPreferencesAsync.');
    final oldPrefs = await SharedPreferences.getInstance();
    final newPrefs = SharedPreferencesAsync();
    final allKeys = oldPrefs.getKeys();
    for (final key in allKeys) {
      final value = oldPrefs.get(key);
      if (value is String) {
        await newPrefs.setString(key, value);
      } else if (value is int) {
        await newPrefs.setInt(key, value);
      } else if (value is double) {
        await newPrefs.setDouble(key, value);
      } else if (value is bool) {
        await newPrefs.setBool(key, value);
      } else if (value is List<String>) {
        await newPrefs.setStringList(key, value);
      }
      logger.info('Migrated key: $key, value: $value');
    }
  }

  if (Platform.isIOS) {
    logger.info('iOS detected: No migration needed as UserDefaults is used.');
  }
}
```

解説

- \_migrateToV2 内で`SharedPreferences`から`SharedPreferencesAsync/WithCache`への移行を行っています。
- bool フラグでマイグレーションを管理する方法もありますが、今後の拡張性を考慮し 簡易的な version でマイグレーションを管理できるようにしています。
- iOS では`UserDefaults`を引き続き使用してるため、マイグレーション処理は行っておりません。

# 他の方法

現在のところ、移行は手動で行う必要がありますが、自動化ツールの開発が進行しているようです。
こちらのツールが完成次第、移行作業を行なっても良さそうですね。
詳しくは以下の Github Issue を確認してください。

https://github.com/flutter/flutter/issues/150732

# 参考

現状の SharedPreference 関連について、とてもわかりやすくまとめられてておすすめです。
https://medium.com/@rishad2002/from-old-to-new-transform-your-flutter-storage-strategy-with-sharedpreferencesasync-and-15b500f40088
