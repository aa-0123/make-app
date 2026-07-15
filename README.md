# make-app
# 出席確認アプリ

学籍番号と学生証の番号をシステムに関連付け、授業時間割を表示するアプリです。

## 主な機能
- 学籍番号・学生証番号の入力とバリデーション
- 月曜〜土曜、1〜6限の授業時間割の自動表示
- レスポンシブデザイン対応

## ユースケース図
- usecaseDiagram
    actor 学生 as "👤 学生 (S01〜S03)"
    actor デモ学生 as "🤖 テスト用学生 (1)"
    actor 教員 as "👨‍🏫 教員 (T01/マスター権限)"

    学生 --> UC_Login_Student : サインインする
    学生 --> UC_Edit_Timetable : 個人時間割をカスタマイズする
    学生 --> UC_View_Attendance : 出席進捗を確認する
    学生 --> UC_Card_Touch : 学生証で疑似打刻する
    学生 --> UC_Request_Help : 救済申請を出す (当日のみ)
    学生 --> UC_Receive_Notification : 個人通知・メール履歴を見る

    デモ学生 --> UC_Card_Touch : 疑似打刻する (クイック対応)

    教員 --> UC_Login_Teacher : サインインする
    教員 --> UC_Switch_View : 学生ポータルへ切り替える (マスター)
    教員 --> UC_View_Detail_Karte : 個別学生カルテを照会する
    教員 --> UC_Control_Table : 出席手動修正・一括保存を行う
    教員 --> UC_Approve_Request : 救済申請を承認・却下する
    教員 --> UC_Send_Mail : 個別・全体メール・掲示板送信を行う
    教員 --> UC_Add_Subject : 新規開講科目を追加する
    教員 --> UC_Set_Rules : 突き合わせ基準ルールを変更する
    教員 --> UC_Quick_Swap : ログアウトなしで役割を切り替える
    
    %% 関係性の拡張
    UC_Switch_View ..> UC_View_Attendance : <<閲覧確認>>
    UC_Switch_View ..> UC_Card_Touch_Proxy : <<代理疑似打刻>>
    UC_Card_Touch_Proxy ..> UC_Card_Touch : デモ学生(1)のみ限定

## クラス図
classDiagram
    class User {
        +String userId
        +String name
        +String password
        +login(role)
    }

    class Student {
        +String cardId
        +Map touchLog
        +saveCustomTimetable()
        +executeTouch()
        +applyHelpRequest()
    }

    class Teacher {
        +changeMatchingRule(rule)
        +sendCustomMail(target, title, body)
        +resolveHelpRequest(reqId, decision)
        +bulkOverrideAttendance()
    }

    class Timetable {
        +List periods
        +Map schedule
        +render()
    }

    class AttendanceData {
        +Map globalAttendanceData
        +initStudentAttendance()
        +getAttendanceRate(studentId, subject)
        +setStatus(studentId, subject, round, status)
    }

    class Application {
        +int id
        +String userId
        +String subjectName
        +String reason
        +String status
    }

    class Notification {
        +int id
        +String title
        +String body
        +String timestamp
        +String targetId
    }

    User <|-- Student
    User <|-- Teacher
    Student "1" -- "1" Timetable : 保持する
    AttendanceData "1" *-- "*" Student : の出席進捗を管理
    Teacher "1" ..> AttendanceData : 手動上書き修正
    Student "1" --> "*" Application : 申請する
    Teacher "1" --> "*" Application : 審査する
    Teacher "1" --> "*" Notification : 配信する
    Student "1" --> "*" Notification : 受信する

## 協調図

graph TD
    %% アクター・オブジェクト定義
    StudentObj["👤 Student (学生ポータル)"]
    SystemObj["⏱ System (スマート出席管理JS)"]
    TeacherObj["👨‍🏫 Teacher (教員ポータル)"]

    %% 学生打刻シナリオ
    StudentObj -- "1. executeStudentTouch(card, password)" --> SystemObj
    SystemObj -- "2. setStatus('S01', '仮出席')<br>3. sendAutoNotification('仮出席検知')" --> StudentObj

    %% 教員確認・不正判定シナリオ
    TeacherObj -- "4. loadTeacherControlTable()<br>5. detectUnmatchedLog('仮出席', '紙提出なし')" --> SystemObj
    SystemObj -- "6. renderStudentListWithLogs()" --> TeacherObj
    TeacherObj -- "7. setStatus('S01', '不正出席')<br>8. saveTeacherOverride()" --> SystemObj
    SystemObj -- "9. pushGlobalWarning('不正打刻自動警告')" --> StudentObj

## 状態遷移図
出席判定ステータスの遷移

stateDiagram-v2
    [*] --> 欠席 : 初期状態 (デフォルト)
    
    欠席 --> 仮出席 : 学生証カードタッチ (打刻ログ記録)
    欠席 --> 出席 : 救済申請の承認 / 教員手動指定
    欠席 --> 公欠 : 教員の公欠承認
    
    仮出席 --> 出席 : 教員が紙の回収物(小テスト等)と照合・確定
    仮出席 --> 不正出席 : 提出物が未提出（ピ逃げ検知・ペナルティ）
    仮出席 --> 欠席 : 一括クリア・欠席リセット
    
    遅刻 --> 出席 : 遅刻の取り消し・手動補正
    出席 --> 遅刻 : 教員の手動変更
    出席 --> 不正出席 : 遡り調査で不正確認・書き換え
    
    不正出席 --> 出席 : 正当な理由確認・事後手動救済
    
    出席 --> [*]
    遅刻 --> [*]
    公欠 --> [*]
    不正出席 --> [*]

 救済申請（不携帯届）の状態遷移

 stateDiagram-v2
    [*] --> 未申請 : 初期状態
    
    未申請 --> 申請中 : 学生が理由とパスワードを入力して当日送信
    
    state 申請中 {
        [*] --> 審査待機 : 教員側BOXへプール
        審査待機 --> 照合中 : 教員が手元の提出物とマッチング確認
    }
    
    申請中 --> 承認 : 教員が【承認】ボタンを押下
    申請中 --> 却下 : 教員が【却下】ボタンを押下 (または虚偽発覚)
    
    承認 --> 出席ステータスに反映 : 出席データを「出席」に上書き補正
    却下 --> 欠席維持 : 元のステータスに據え置き
    
    出席ステータスに反映 --> [*]
    欠席維持 --> [*]

　
