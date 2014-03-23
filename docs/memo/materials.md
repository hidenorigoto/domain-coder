# 2014/3/23 ユースケース

リポジトリの役割：CUD(Command) / Query

エンティティクラスにActiveRecord
```
namespace UseCase;

use Entity\User;
use Dict\User as 登録済みユーザー;

class 会員登録 extends UseCase
{
    public function __invoke(User $登録情報)
    {
        登録済みユーザー::add($登録情報);
    }
}

```

エンティティクラスがCollectionクラスを返す。Collection操作がUnitOfWorkに対応。
```
namespace UseCase;

use Entity\User;
use Dict\User as 登録済みユーザー;

class 会員登録 extends UseCase
{
    public function __invoke(User $登録情報)
    {
        登録済みユーザー::collection()->add($登録情報);
    }
}

```

登録時のファクトリ（共通ファクトリ利用）

```
namespace UseCase;

use Entity\User;
use Dict\User as 登録済みユーザー;

/**
 * @UseFactory("UserFactory")
 */
class 会員登録 extends UseCase
{
    public function __invoke(User $登録情報)
    {
        登録済みユーザー::collection()->add($登録情報);
    }
}

```

登録時のファクトリ（その場定義）

```
namespace UseCase;

use Entity\User;
use Dict\User as 登録済みユーザー;

class 会員登録 extends UseCase
{
    public function __invoke(User $登録情報)
    {
        登録済みユーザー::collection()->add($登録情報);
    }

    public function factory($data)
    {
        $entity = new 登録済みユーザー();
        $entity->name = $data['name'];
        // ...
        
        return $entity;
    }
}

```





# 2014/3/20 対象システム

記述できる仕様の部品の妥当性を検証するために、サンプルシステムをいくつか用意しておく必要があるだろう。特にユースケースをどう記述できるのか考えなくてはならない。

* 会員登録サンプル

UserRegistrationService#register()

```
    public function register(User $user)
    {
        $user->setActivationKey(base64_encode($this->secureRandom->nextBytes(24)));
        $user->setPassword($this->passwordEncoder->encodePassword($user->getPassword(), User::SALT));
        $user->setRegistrationDate(new \DateTime());

        $this->entityManager->getRepository('Example\UserRegistrationBundle\Domain\Data\User')->add($user);
        $this->entityManager->flush();

        $emailSent = $this->userTransfer->sendActivationEmail($user);
        if (!$emailSent) {
            throw new \UnexpectedValueException('アクティベーションメールの送信に失敗しました。');
        }
    }
```

こちらのユースケース（シナリオで）は、
* メインコース：ユーザーエンティティの登録
* 補助コース：メールの送信
となっている。




UserRegistrationService#activate()

```
    public function activate($activationKey)
    {
        $user = $this->entityManager->getRepository('Example\UserRegistrationBundle\Domain\Data\User')->findOneByActivationKey($activationKey);
        if (is_null($user)) {
            throw new \UnexpectedValueException('アクティベーションキーが見つかりません。');
        }

        if (!is_null($user->getActivationDate())) {
            throw new \UnexpectedValueException('ユーザーはすでに有効です。');
        }

        $user->setActivationDate(new \DateTime());
        $this->entityManager->flush();
    }
```




# 2014/3/19 REA

REA（Resource、Event、Agent）会計システムのための分析フレームワーク

* [REAとビジネスパターン入門(4)](http://itpro.nikkeibp.co.jp/article/Watcher/20070925/282886/)

ドメインルールと言って、例えば●の増加には☓の現象が必ず対応する、というようにドメインの要素間にルールが存在し、これによりモデルが検証できる。（導かれる）
もともと会計システムを念頭に置いているため、帳簿組織論とスムーズにマッチしそうではある。

レイヤーや振る舞いの基礎部品などは参考になりそうだ。
ドメイン駆動設計とのマッチングについてはまだわからない。
REAのリソース、イベントについてはわかるが、エージェントについては、どういう役割なのか調べないとわからないものもある。


# 2014/3/18 仕様記述について

Alloyについて少し調べた。以下の書籍を購入。

* [抽象によるソフトウェア設計－Alloyではじめる形式手法](http://www.amazon.co.jp/dp/4274068587)

Alloyは形式手法による仕様記述（抽象モデルの記述）を行える言語（とその検証環境）。
Alloyを使って、モデルの静的な側面（状態）と動的な側面（操作：制約）を記述していく。

第２章の例

```
module tour/addressBook1

sig Name, Addr {}
sig Book {
    addr: Name -> lone Addr
    }

pred show (b: Book) {
    #b.addr > 1
    #Name.(b.addr) > 1
    }

pred add (b, b': Book, n: Name, a: Addr) {
    b'.addr = b.addr + n -> a
    }
```

Name、Addr、Bookという3つのシグネチャがあり、それぞれがオブジェクトの集合を表す。
制約は集合の1つの状態に対して記述（上の例での show）したり、集合の2つの状態の関係として記述（上の例での add）したりできる。
後者の書き方が操作の記述ということになる。

    手続き型言語に慣れていて、モデリング言語は初めてという人には、操作をこんなふうに記述することが奇妙に思えるかもしれない。この記述には状態の変化が明示的に書かれていない。代わりに、操作前後でのアドレス帳の状態が異なる名称（bとb''）で与えられ、操作が及ぼす影響はそれら2状態の間に成り立つ性質として記述されている。
    抽象によるソフトウェア設計 2.2 動的な側面：操作を追加する p. 10

上記の例をOOP言語（DDD）で解釈してみると、

* Name、Addr、Bookは、エンティティの集合なので、エンティティクラス定義にあたると思われる
* showはエンティティの集合に対する制約で、集約ルートにおける条件検査か、何らかのバリデーション
* addは、エンティティの集合に対する操作で、つまりリポジトリ操作にあたると思われる



