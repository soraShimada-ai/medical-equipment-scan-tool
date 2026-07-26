【システム詳細仕様書】眼科医療機器スキャン・自動レポートアプリ

1. システム概要・背景

1.1 背景・目的

医療機器営業において、訪問先眼科内に設置されている機器情報の調査・管理は手作業が多く、負荷がかかっています。 本アプリは、スマホカメラで医療機器のバーコードやラベルを読み取り、現在地のGPS情報から訪問先眼科名を自動特定したうえで、端末内DBへの保存とメール（Brevo API）による自動通知を瞬時に行うことで、営業現場での情報共有と収集の完全自動化を実現します。

1.2 アプリコンセプト

「起動即スキャン」: アプリ起動と同時にスキャン画面が立ち上がり、ワンタップも挟まず撮影を開始できる。
「ワンストップ完全自動処理」: スキャン成功と同時に「位置情報取得 → 最寄り眼科判定 → ローカルDB保存 → Brevo経由メール自動送信」を一連の流れとして自動実行する。
「完全0円運用」: オンデバイス処理と各サービスの無料枠を活用し、インフラ費用・API利用料0円で運用する。
2. システム構成・開発環境

対象プラットフォーム: iOS / Android / Webブラウザ（クロスプラットフォーム）
開発フレームワーク: React Native (Expo)
主要ライブラリ:
カメラ・バーコード: expo-camera
OCR（文字認識）: react-native-mlkit-ocr (オンデバイス処理)
位置情報: expo-location
ローカルデータベース: expo-sqlite
メール送信サービス: Brevo (Sendinblue) API（無料枠：最大300通/日）
3. 機能要件一覧

機能ID	機能名	詳細・要件
F-01	即時スキャン機能	アプリ起動時にカメラを自動起動。バーコード（1D/2D）およびラベルテキスト（型番・メーカー・製造年）を自動検知。
F-02	位置情報・日時取得	スキャン実行時の日時（YYYY-MM-DD HH:mm:ss）およびGPS位置情報（緯度・経度）を自動取得。
F-03	最寄り眼科自動特定	事前登録された「顧客眼科マスター」と現在地GPSの距離を判定し、最も近い眼科名を自動抽出。
F-04	ローカルDB自動保存	読み取りデータ・日時・位置情報・特定された眼科名を端末内SQLiteに即座に保存。
F-05	メールバックグラウンド自動送信	Brevo REST APIをバックグラウンドで呼び出し、設定済みアドレスへスキャン内容を自動送信（標準メールアプリは介さない）。
F-06	履歴一覧・詳細表示	ローカルDBに保存された過去のスキャン履歴を時系列で閲覧・検索可能。
F-07	設定管理	通知先メールアドレスおよびBrevo APIキーの設定・変更。
4. データベース仕様（ローカルDB）

4.1 スキャン履歴テーブル (equipment_scans)

カラム名	論理名	データ型	制約	補足・例
id	ID	INTEGER	PRIMARY KEY AUTOINCREMENT	主キー
scanned_at	スキャン日時	DATETIME	NOT NULL	2026-07-26 14:30:00
latitude	緯度	REAL	NOT NULL	33.5902
longitude	経度	REAL	NOT NULL	130.4207
location_name	眼科名	TEXT	NOT NULL	「博多駅前アイクリニック」
barcode_number	バーコード番号	TEXT	NULL可	4987123456789
model_number	型番	TEXT	NOT NULL	XYZ-1000
manufacturer	製造メーカー	TEXT	NOT NULL	〇〇メディカル
manufacture_year	製造年	TEXT	NULL可	2023
created_at	登録日時	DATETIME	DEFAULT CURRENT_TIMESTAMP	システム登録日時
4.2 顧客眼科マスターテーブル (clinics)

カラム名	論理名	データ型	制約	補足・例
id	ID	INTEGER	PRIMARY KEY AUTOINCREMENT	主キー
name	眼科名	TEXT	NOT NULL	「博多駅前アイクリニック」
address	住所	TEXT	NULL可	「福岡市博多区博多駅前2-1-1」
latitude	緯度	REAL	NOT NULL	33.5902
longitude	経度	REAL	NOT NULL	130.4207
デモデータ挿入用 SQL (初期流し込み用)

CREATE TABLE IF NOT EXISTS clinics (
id INTEGER PRIMARY KEY AUTOINCREMENT,
name TEXT NOT NULL,
address TEXT,
latitude REAL NOT NULL,
longitude REAL NOT NULL
);

INSERT INTO clinics (name, address, latitude, longitude) VALUES
('博多駅前アイクリニック', '福岡市博多区博多駅前2-1-1', 33.5902, 130.4207),
('天神中央眼科', '福岡市中央区天神1-10-1', 33.5916, 130.4017),
('西新みらい眼科クリニック', '福岡市早良区西新4-1-1', 33.5832, 130.3582),
('香椎駅前眼科外科', '福岡市東区香椎駅前1-8-1', 33.6593, 130.4438),
('久留米総合眼科', '福岡県久留米市東町31-1', 33.3168, 130.5181);

5. 主要処理ロジック仕様

5.1 最寄り眼科特定ロジック（Haversine公式）

GPSから取得した緯度経度と clinics テーブル内の全データの距離を計算し、最短距離にある施設を特定します。
export interface Clinic {
id: number;
name: string;
address: string;
latitude: number;
longitude: number;
}

// 2点間の球面距離(メートル)算出処理
function getDistanceMeters(lat1: number, lon1: number, lat2: number, lon2: number): number {
const R = 6371e3; // 地球の半径 (m)
const φ1 = (lat1 * Math.PI) / 180;
const φ2 = (lat2 * Math.PI) / 180;
const Δφ = ((lat2 - lat1) * Math.PI) / 180;
const Δλ = ((lon2 - lon1) * Math.PI) / 180;

const a =
Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
Math.cos(φ1) * Math.cos(φ2) * Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

return R * c;
}

export function findNearestClinic(currentLat: number, currentLon: number, clinics: Clinic[]): Clinic | null {
if (clinics.length === 0) return null;

let nearest = clinics[0];
let minDistance = getDistanceMeters(currentLat, currentLon, nearest.latitude, nearest.longitude);

for (let i = 1; i < clinics.length; i++) {
const distance = getDistanceMeters(currentLat, currentLon, clinics[i].latitude, clinics[i].longitude);
if (distance < minDistance) {
minDistance = distance;
nearest = clinics[i];
}
}

return nearest;
}

5.2 メール自動送信処理（Brevo API）

Brevoの v3/smtp/email エンドポイントを呼び出し、自動送信を行います。
export async function sendScanReportEmail(params: {
apiKey: string;
toEmail: string;
clinicName: string;
scannedAt: string;
lat: number;
lng: number;
barcode: string;
modelNumber: string;
manufacturer: string;
manufactureYear: string;
}) {
const url = 'https://api.brevo.com/v3/smtp/email';

const bodyData = {
sender: { name: "機器スキャンアプリ", email: "noreply@yourdomain.com" },
to: [{ email: params.toEmail }],
subject: `【機器登録通知】${params.clinicName} - ${params.manufacturer} ${params.modelNumber}`,
textContent: `
医療機器のスキャンデータが登録されました。

【スキャン情報】
・スキャン日時：${params.scannedAt}
・スキャンした場所：${params.clinicName}
・位置情報（緯度/経度）：${params.lat}, ${params.lng}

【医療機器情報】
・バーコード番号：${params.barcode}
・型番：${params.modelNumber}
・製造メーカー：${params.manufacturer}
・製造年：${params.manufactureYear}

-----------------------------------------
※本メールは医療機器スキャンアプリより自動送信されています。
`
};

const response = await fetch(url, {
method: 'POST',
headers: {
'accept': 'application/json',
'api-key': params.apiKey,
'content-type': 'application/json'
},
body: JSON.stringify(bodyData)
});

return response.ok;
}

6. メール送信フォーマット仕様

件名

【機器登録通知】[スキャンした場所] - [製造メーカー] [型番]

本文テンプレート

医療機器のスキャンデータが登録されました。

【スキャン情報】
・スキャン日時：2026/07/26 14:30:00
・スキャンした場所：博多駅前アイクリニック
・位置情報（緯度/経度）：33.5902, 130.4207

【医療機器情報】
・バーコード番号：4987123456789
・型番：XYZ-1000
・製造メーカー：〇〇医療機器株式会社
・製造年：2023年

-----------------------------------------
※本メールは医療機器スキャンアプリより自動送信されています。

7. 費用・運用ガイドライン

API利用量の管理: Brevoの無料枠は1日300通まで。営業1名あたりの1日のスキャン回数は十分カバー可能。
オンデバイス処理の徹底: OCRや場所の特定（Haversine式）はすべて端末内で完結するため、外部クラウド（Google Cloud VisionやGoogle Maps API）への依存および従量課金リスクはゼロ。
こちらの仕様書をベースに、いつでも開発に着手することができます！ 画面のデザイン案やモックアップの作成に進む場合は、お気軽にお知らせください。

