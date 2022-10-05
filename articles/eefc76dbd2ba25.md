---
title: "【Flutter】アプリ全体のアーキテクチャを0から考えて作り直した話"
emoji: "🖼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Dart", "アーキテクチャ", "設計"]
published: true 
---

ここ半年ほど、仕事で Flutter アプリを 0 から作り直しています。

ちょうど今年の個人的なテーマを「アーキテクチャ」に据えていたこともあり[^1]、またその一環として [「Clean Architecture 達人に学ぶソフトウェアの構造と設計」](https://www.amazon.co.jp/dp/4048930656) （以下：クリーンアーキテクチャ本）を読んでいたこともあり、この作り直しでは「アーキテクチャ」をしっかりと自分の頭で考えながら作ろうと決めて取り組んできました。

アーキテクチャについて頭を悩ませながら実装を進めること約半年、ようやくアプリが形になるとともにある程度知見も溜まってきましたので、その知見を一般化した内容をこの記事にまとめていきたいと思います。

# 注意

この記事は、「Flutter アプリのアーキテクチャはこれがベストプラクティス！」という類の記事ではありません。あくまで __私の目の前の要件ではこれが最適と判断した__ という一例の紹介になります。

ここに書く内容はどのようなアプリでもそのまま適用してうまくいくようなものではありません。クリーンアーキテクチャ本でも強調されている（と私は理解している）ように、 __それぞれのソフトウェアの開発者がそれぞれの要件や状況を考慮した上でその時点での最適なアーキテクチャを考え続ける__ のが大事だと思っています。

「次のアプリはここに書かれたパターンに当てはめて作ってみよう」と考えるのではなく、「ってことは自分のアプリだとこうするのが良さそうかな？」と考えながらこの記事を読んでいただければと思います。この記事で紹介するアーキテクチャも、一般的に紹介されている何か具体的なパターンに必要以上に引っ張られないように意識して考えています。[^2]

# アプリの主な要件
アーキテクチャの具体的な話に入る前に、私が開発中のアプリがどのような要件なのかを説明します。具体的なアプリ名や内容までは書けないので、アーキテクチャの検討に強く影響している要素を列挙します。

### バックエンド（今回は Firestore）に保存したデータを取得して画面に表示するパターンが多め

いわゆる「JSON に色を付ける」程度で事足りる機能が 6 割程度を占めます。ただし、いくつかの機能では複数のコレクションからデータを引っ張ってマージしたりする必要があったり、取得したデータをアプリ内のロジックに従って一度料理してから UI に表示するような機能もあります。逆に保存すべきデータをアプリ内のちょっとした処理で生成しなければならないような機能もあります。

つまり、データをそのまま入出力する「だけ」とは言えないものの、一方で込み入ったシステムがアプリ内に必要なわけでもない、という温度感です。言い換えると、Widget クラスと Firestore の操作クラスを直接繋ぐとその間のロジックが大変なことになります。

### 「似たような機能を持った別アプリ」の開発・リリースを念頭におく

今回開発したアプリをベースに、ターゲットやデザインコンセプトを変更した別バージョンのアプリや、一部の機能のみを抽出して特化させたアプリの開発をビジネス的な方針として検討されている、というのも大事な要件のひとつです。そのため、コードは可能な限り共通利用できる形で設計することが求められます。

### GPS を利用するため机上で開発・デバッグしづらい

このアプリは GPS を利用した機能を中心としています。そのため、机上の開発には GPS をエミュレートする機能が必要になるのですが、Android / iOS 各プラットフォームに標準で用意されているエミュレート機能を利用するにはそれぞれに一手間が必要だったり制約があったりします。[^3]

そのため、任意のロジックに従ってダミーの位置情報する仕組みと、それをビルド時の設定等で簡単に切り替えられる仕組みが必要です。

---

というようなアプリであることを念頭に読み進めていただければと思います。

# 全体を 3 つのレイヤー +α に分け、「境界」と「依存関係」を整理する

まずは大枠から考えます。クリーンアーキテクチャ本を読んでまず自分が学んだのは、 __レイヤーの「境界」を定義し、レイヤーの「依存関係」を整理する__ ことです。

以下の図が今回のアプリの大枠となるアーキテクチャです。境界がどこにあるのか、それぞれのレイヤーの依存関係がどうなっているのかに着目して見ていただければと思います。

![アーキテクチャ概要](https://user-images.githubusercontent.com/20849526/193012089-11593262-b45a-408d-84d7-f6f7a9f91f47.png)

図にするととてもシンプルですね。どこかで見たような分け方と感じるのではないかと思います。重要なのは、それぞれのレイヤーに設定するルールと、依存関係に対するルールを適切に決めて守ることです。

# ルールを定める

それでは、先述したルールがどのようなものなのかを確認していきましょう。その後、そのルールによって生まれるメリットを整理していきます。

## 依存関係のルール

まずは依存関係についてです。「依存関係」については、言葉で長々と説明するよりも具体的なコード例を使って説明を進めます。

たとえば、「グループ内のメンバー一覧を取得して画面に表示する」ような機能をイメージするとき、以下のコードは View レイヤーに含まれる `MembersState` クラスが BusinessLogic レイヤーに含まれる `MembersLogic` クラスを呼び出している（参照している）ため、「`MembersState` は `MembersLogic` に依存している」と説明できます。

```dart:view/member_state.dart
import 'package:myapp/businesslogic/member_logic.dart';

class MemberState extends State<MemberScreen> {
  final logic = MemberLogic();

  // 画面表示に利用するメンバー一覧
  late List<Member> _members;

  @override
  void initState() {
    // MemberLogic を通してデータを取得
    _members = logic.getMembers();
  }
}
```

最初の図を見ると、View レイヤーと BusinessLogic レイヤーの依存関係は __View -> BusinessLogic__ と決めているため、上記のコードは OK です。

一方で、View レイヤーと Repository レイヤーの間には矢印がないため、以下のコードは NG となります。

```dart:view/member_state.dart
import 'package:myapp/repository/firestore_member_repository.dart';

class MemberState extends State<MemberScreen> {
  final repository = FirestoreMemberRepository();

  // 画面表示に利用するメンバー一覧
  late List<Member> _members;

  @override
  void initState() {
    // MemberLogic を通してデータを取得
    _members = repository.fetchMembers();
  }
}
```

このように「どのレイヤーはどのレイヤーに依存してよいか（参照してよいか）」をアーキテクチャのルールとして定めています。

ちなみにコードがこのルールに従っているかどうかは、import 文を見れば一発でわかります。

__View -> BusinessLogic__ の場合、import 文に並ぶファイルは必ず `businesslogic/xxxx.dart` となるはずです。一方で `repository/xxx.dart` が import 文に含まれる場合、それは __View -> Repository__ の依存関係ができてしまっていることになるため、機械的に NG と判断できます。

これを静的に解析するために、今回は `import_lint` というパッケージを導入してみています。

https://pub.dev/packages/import_lint

## 依存関係の逆転

さて、 __View -> BusinessLogic__ のように、呼び出しの経路がそのまま依存の方向になっている場合は話がシンプルなのですが、BusinessLogic と Repository の関係をみてみると、__BusinessLogic が Repository を呼び出したい__ にもかかわらず、依存関係は __Repository -> BusinessLogic__ となっています。

このように「呼び出し」の関係と「依存」の関係を逆転させてルールづけている理由については後述しますが、ここでは「依存関係の逆転」という考え方を使って依存の方向性を整理する手法について先に説明します。

まず、先ほどのコード例をみるとわかる通り、「BusinessLogic が Repository を呼び出す」をそのまま実装しようとすると依存関係は __BusinessLogic -> Repository__ となってしまいます。

これを「逆転」させるには、インターフェースを BusinessLogic レイヤーの中に定義し、`MemberLogic` クラスはそのインターフェースにのみ依存するようコーディングします。

```dart:businesslogic/interface/member_repository.dart
// メンバーデータにアクセスするためのインターフェース
abstract class MemberRepository {
  Future<List<Member>> fetchMembers;
}
```

```dart:businesslogic/member_logic.dart
import 'package:myapp/businesslogic/interface/member_repository.dart';

class MemberState extends State<MemberScreen> {
  MemberState(this.repository);

  final MemberRepository repository;

  Future<List<Member>> getMembers() async {
    return await repository.fetchMembers();
  }
}
```

次に、Repository レイヤーには `MemberRepository` を継承した `FirestoreMemberRepository` クラスを実装します。

```dart:repository/firestore_member_repository.dart
import 'package:myapp/businesslogic/interface/member_repository.dart';

// Firestore にアクセスしてメンバーデータを出し入れするクラス
class FirestoreMemberRepository extends MemberRepository {
  @override
  Future<List<Member>> fetchMembers() async {
    // Firestore の所定のコレクションからメンバーデータを取得してくる処理。
  };
}
```

このようにすることで、import 文に着目すると確かに BusinessLogic レイヤーの `MemberLogic` クラスは同じレイヤー内のファイルにのみ依存していて、一方で Repository レイヤーの `FirestoreMemberRepository` クラスは BusinessLogic レイヤーのファイルに依存する（ __Repository -> BusinessLogic__ ）という、呼び出しの方向と依存の方向の「逆転」が見てとれるでしょう。

「なんだ、インターフェースとは言え `Repository` の名前がついたファイルを無理矢理 `businesslogic` フォルダに置いただけじゃないか、ただの言葉遊びじゃないか。」と思われる方もいるかもしれませんが、[^4]このテクニックを使って依存関係を整理することで明確なメリットが生まれます。次はそれについてみていきましょう。

## 依存関係を整理するメリット

今回のアーキテクチャにおいて、依存関係をキッチリ整理するメリットのひとつが「交換可能性」です。

その端的な例が BusinessLogic レイヤーで、図をみると BusinessLogic レイヤーは（`Data` を除いて）他のどのレイヤーに対しても矢印が向かっていません。つまり、他のレイヤーにどのような変更があったとしても、極端な話 __View レイヤーと Repository レイヤーをフォルダごと削除したとしても BusinessLogic レイヤーにはコンパイルエラーは発生しない__ ということです。

これは以下の 2 つの要件を満たすのに役立ちます。

1. 「似たような機能を持った別アプリ」を開発する
2. データ取得元（特に GPS）をエミュレートする

それぞれ詳しくみていきましょう。

### 1. 「似たような機能を持った別アプリ」を開発する 

たとえば __「UI をガラッと変えて機能も絞り込んだ別アプリを開発したい」__ という話がビジネス的に持ち上がった場合、開発者は「じゃあこのクラスとコレとコレを共通化して、あっちは少し切り離して、、ああこのクラスも一緒にリファクタしなきゃなのか、、」といったアレコレを考える必要はありません。__機械的に `businesslogic` フォルダをコピー & ペーストし[^5]、その中の必要なクラスだけを呼び出す新しい View レイヤーを作ればよいだけ__ です。

またその際、データの保存先やデータ形式が同じ場合は Repository レイヤーも一緒にコピペできます。逆にもし「今回のアプリは Firebase じゃなくて AWS で API 用意するからそれ叩いてよ」となった場合は Repository レイヤーを実装し直すだけで BusinessLogic レイヤーはやっぱり何も変更せずに使いまわせます。

### 2. データ取得元（特に GPS）をエミュレートする

別アプリを作る場合以外でも、たとえば「実行モードによってデータの保存・取得先を変更する」ような要件にも対応しやすくなります。

たとえば __「Firestore にちゃんと接続して UI からデータベースまで通して動作確認するモード」と「Firestore の都合に左右されずに UI を開発したいからメモリ上だけでデータを出し入れする簡易モード」を切り替えたい__ 場合を考えてみましょう。コードとしては、 `FirestoreMemberRepository` と `InMemoryMemberRepository` の 2 つクラスを Repository レイヤーの中に作ることになります。

そのどちらのクラスを利用するかを実行時の引数（たとえば `--dart-define` など）で切り替えることを考えたとき、アーキテクチャの図をみてもわかるとおり Repository レイヤーに向かう矢印はどこにも存在しないため、View や BusinessLogic など他のレイヤーを考慮する必要は一切ありません。「このモードの場合はロジックのここに影響がでちゃうからここも一緒に if で分岐して、、ああやっぱりリファクタ必要かも、、」と悩む必要はないのです。

同じ要領で GPS もエミュレート可能です。「端末の GPS から位置情報を取得するクラス」と「ロジックに従ってランダムな位置情報を返却するクラス」、もしくは「過去のデータを取得して移動を再現するクラス」を Repository レイヤーに用意したら実行時の引数に従って切り替えれば良いだけです。View や BusinessLogic はその緯度経度がどこから発生したものなのかを気にする必要はありません。[^6]

---

このように、依存関係を整理することで、どこが切り離し可能で、その際どこに影響が出るのか（特に「コンパイルエラー」という形で）が予測しやすくなり、それによって別アプリの開発やちょっとしたエミュレート機能の作り方がイメージしやすくなるのがとても大きなメリットだと感じています。

## レイヤーごとのルール

さて、レイヤーを分け依存関係を整理できたところで、次はそれぞれのレイヤーにどのようなコードを実装するのかについてのルールをみていきましょう。

個別の説明に入る前に、先ほどの図に Flutter や Firebase など自分たちが書くコード以外の関係性も追記してみましょう。

![より詳しいアーキテクチャ](https://user-images.githubusercontent.com/20849526/193082591-fcda0c03-0d76-4212-b15a-359ebfcee7be.png)

こちらを見るとわかる通り、__Flutter に依存してよいのは View だけ、Firebase に依存してよいのは Repository だけ__ 、となっています。このことを念頭に読み進めていただければと思います。

### BusinessLogic

まずは依存関係の中心となる BusinessLogic レイヤーです。

BusinessLogic には、その名の通り「ユーザーのやりたいこととその手順」をコードで表現します。[^7]これだけだと具体的ではないので説明を加えると、「どんなプラットフォームで動作するシステムであっても、なんならシステムですらなくてもユーザーがやりたいこと」をやるのがこのレイヤーと考えています。

たとえば、「自分の現在地から半径300mの中にあるラーメン屋を近い順にリストアップしたい」という「やりたいこと」があった場合、その手順を言葉で表すと以下の通りです。

1. 日本中のラーメン屋をすべてリストアップして
2. 自分の現在位置を調べて
3. その一件一件が現在位置から何メートル離れているかを計算して
4. 計算結果が300m以内だったらそのラーメン屋をどこかにメモしておいて
5. 最後まで調べ終わったらメモを見返して
6. 近い順に並べ替えたメモを作ったら完成

となります。

この作業は（時間や能力的な制約を考えなければ）アプリでもPCでも、GUI でも CUI でも、コンピューターでも手作業でもできると言えるのではないでしょうか。そのような作業手順をコードで書き表すのが BusinessLogic レイヤーです。

言い換えると、このレイヤーのコードには「Flutter を使う」ことも「Firebase からデータを取得する」ことも書いてはいけません。極端な話、 __一切のパッケージを利用せずに Dart の標準クラスのみで実装する__ ことを目指すレイヤーであると言っても過言ではありません。

例えば、上の例を実現する `RamenListLogic` クラスのコードは以下のようなイメージになるでしょう。（あくまでイメージです）

```dart:ramen_list_logic.dart
import 'package:myapp/businesslogic/interface/ramen_repository.dart';
import 'package:myapp/businesslogic/interface/location_repository.dart';
import 'package:myapp/data/restaurant.dart';

class RamenListLogic {
  RamenListLogic(this.ramenRepository, this.locationRepository);

  final RamenRepository ramenRepository;
  final LocationRepository locationRepository;

  Future<List<Restaurant>> getVisitableRestaurants() async {
    final allRestaurants = await ramenRepository.fetchAll();
    final currentLocation = await locationRepository.detectCurrentLocation();
    final visitableRestaurants = allRestaurants.where((restaurant) {
      return _calcDistance(restaurant.location, currentLocation) <= 300;
    }).toList();
    return visitableRestaurants..sort((r1, r2) => r1.distance - r2.distance);
  }
}
```

上記のコードのポイントは以下のとおりです。

- `view` や `repository` フォルダ内のファイルを import しない
- Flutter や Firebase が提供するファイルをインポートしない
- `RamenRepository` や `LocationRepository` の具象クラスは外から受け取る（今回はコンストラクタで）

これによって「他のレイヤーに一切依存しない（Data はちょっと例外として）」「ピュアな Dart コードのみで実装された」クラスが完成します。

このとき、Repository のインターフェースには __BusinessLogic を表現する上で最も素直に利用できる形のメソッド__ を定義しています。個人的な BusinessLogic レイヤーのイメージは __他の何も気にすることなく最もワガママに実装できるレイヤー__ です。何かアプリの機能を実装する、となった際はまずここからスタートするように意識しています。Firestore 上のデータ形式がこうだからとか、画面遷移がこうなっているからとか、そういったことは一切考えてはいけません。

こうすることで、単体テストや動作確認がとても楽になります。このレイヤーのコードは Flutter に依存しないため `flutter run` でアプリを動かすことなく `dart` コマンドでサクッと実行できますし、単体テストであれば `dart test` で動作確認が可能です。[^8]

コンストラクタに渡すべき Repository クラスもテストしやすい適当なモックを作れば OK でしょう。

```dart:mock_ramen_repository.dart
import 'package:myapp/businesslogic/interface/ramen_repository.dart';

class MockRamenRepository extends RamenRepository {

  /// 適当に固定データを返却する
  @override
  Future<List<Restaurant>> fetchAll() async {
    return const [
      Restaurant(...),
      Restaurant(...),
      Restaurant(...),
      Restaurant(...),
      Restaurant(...),
      Restaurant(...),
    ]
  }
}
```

### Repository

Repository レイヤーは __データベースにおいて発生するすべての諸事情を吸収する__ のが役割です。

諸事情とは、たとえば以下のようなものがあります。

- データベースには Firestore を利用する
- でも部分的に Functions を叩いて取得する
- 目的のデータを構築するために複数のコレクションからデータを引っ張ってくる必要がある
- 機能追加に伴ってちょっとデータ構造的に破綻する箇所が出てきたから、データ構造や名前を変更する
- こっちの派生アプリでは大人の事情で AWS を使う

ワガママな BusinessLogic が指定したインターフェースに従ってデータを返却できるよう、データの取得方法と変換を行うコードをここに書いていくと考えるとよいでしょう。

理想的な姿は、バックエンドにどのような事情が発生しようとも、View や BusinessLogic レイヤーのコードを一切変更することなく解決できることです。

そのため、たとえば Firestore パッケージが用意する `DocumentSnapshot` のような型のまま `BusinessLogic` レイヤーにデータを返却することはできません。かならず `Data` レイヤーで自分が定義した型に変換してから返却します。View や BusinessLogic のファイルに Firestore の import 文が見えた瞬間、それは何かが間違っていると判断します。

### UI

UI はおそらく一番イメージしやすいのではないでしょうか。

Figma を見ながら Flutter の書き方に従って Widget クラスを定義し、`provider` や `riverpod` などの状態管理パッケージでリビルドを制御するコードを書くのがこのレイヤーです。

BusinessLogic レイヤーに用意したクラスをどのように利用するかは設計次第です。私が開発するアプリでは `StatefulWidget` の `State` クラスが BusinessLogic レイヤーのオブジェクトを使う場合もありますし、アプリのさまざまな場所で共有したいデータは `provider` を使ってひとつの状態オブジェクトを共有します。時と場合に応じて適切なものを使うとよいでしょう。[^9]

なお、クリーンアーキテクチャ本がおそらく想定している CLI アプリケーションにおいては、入力と出力は別々のコンポーネントが担当することになっているように読みましたが、アプリなどの GUI アプリケーションにおいては __入力するコンポーネントと出力するコンポーネントは同じ__ です。どちらも Widget クラスの `build()` メソッドに画面の表示内容もユーザーの操作も書かなければならないことを考えるとイメージしやすいと思います。

### main.dart

最後に大事なのが `main.dart` です。クリーンアーキテクチャ本にも書いてある通り、__全ての汚れ役は Main が担当__ します。

今回のアプリでいう「汚れ役」とは、たとえば `--dart-define` の値に応じた Repository レイヤーの具象クラスの切り替えが挙げられます。

今回は [get_it パッケージ](https://pub.dev/packages/get_it) を利用し、`--dart-define` の値に応じて UI から `GetIt.I<RamenRepository>()` のように呼び出した時に受け取れる具象クラスが切り替わるようにしています。

```dart:service_locator.dart
void prepare() {
  final mode = String.fromEnvironment('REPOSITORY_MODE');

  if (mode == 'server') {
    GetIt.I.registerSingleton<RamenRepository>(
      FirestoreRamenRepository(),
    );   
  } else {
    GetIt.I.registerSingleton<RamenRepository>(
      MockRamenRepository(),
    );   
  }
}
```

Firebase エミュレーターも利用したりしているため、エミュレーターを利用するための設定（`FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080)` の呼び出しなど）もここで行います。

各レイヤー内に綺麗に閉じ込められない処理（主に準備処理）はここにまとめちゃうイメージです。`main.dart` にレイヤーや依存関係といった概念は存在しません。

# まとめ

各レイヤー内のさらに詳しい設計など、書きたいことやおそらくみなさんが気になるであろうことはまだまだありますが、一旦長くなってしまったのでこの記事は以上とさせていただければと思います。

まだまだアーキテクチャについては勉強し始めですが（ちょっと前まで "DDD" がなんの略なのかもしらなかった）、まずはクリーンアーキテクチャ本の教えに従って「目の前の要件やプロジェクトの状況に応じた最適なアーキテクチャ」を常に考え続けることをこれからも継続していきたいと思います。

# 続き

実際に分割したそれぞれのレイヤーについて、詳しい説明を別の記事に書きました。続けて読んでみていただければと思います。

https://zenn.dev/chooyan/articles/17dde307509248

[^1]: 去年は「Flutter の内部実装」でした。

[^2]: ですので、「この設計、〇〇パターンのこの原則に違反してるじゃん」「この用語、△△パターンでの定義と違うじゃん」というような批判はご遠慮ください。そもそもそのようなパターンに準拠することを目的としていません。

[^3]: 実機では使えない、任意の位置情報をシミュレートできない、など。

[^4]: 自分は思いました。

[^5]: ちゃんとやるならパッケージ化も検討します。

[^6]: アーキテクチャについて考える前は「GPS の取得」と言ったらなんとなく View に含まれるのかなあ、と考えたりしていたのですが、よくよく考えると GPS の取得も「データの取得」なので Repository レイヤーに入れるのが自然なことに気がつきました。

[^7]: 「ビジネスロジック」というと「業務ロジック」と訳されがちですが、個人的にはこれは少し直訳すぎるかなと感じています。「業務」という言葉を言葉通りにとらえてしまうと、「じゃあ趣味アプリの場合は『業務』ロジックって存在しないの？」とちょっとした混乱が発生するためです。"business" という単語が、たとえば "It's my business.（それは私のやるべきことです）" や "It's not my business.（そんなん知ったこっちゃないよ、あなたの問題でしょ？）" みたいな使われ方をすることを考えてみると、__「やりたいことを実現するための手順」__ と考えるとイメージしやすいのではないかな、と考えています。

[^8]: `flutter run` でアプリを実行しての動作確認は、いくらホットリロードが便利な Flutter と言えども手間です。実行時にビルドエラーが出ることもありますし、PC のスペック次第ではビルドに数分かかることもありますし、実行できたとしてもお目当てのコードが動く画面まで進めなければなりません。その画面に到達する前に別のエラーが発生する場合もあるでしょう。ログも関係ないものがたくさん出て見づらいです。Dart 単体で動かせるコードというのは想像以上に扱いが楽であることが身に染みています。

[^9]: このあたりの状態管理まわりの使い分けや細かい設計については書き始めると長くなるため、別記事でまとめられればと思います。