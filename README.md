# OpenFOAM非ニュートン粘性モデル カスタマイズプロジェクトのために改造手順（ライブラリ追加式）
##### 2015年8月14日オープンCAE勉強会＠富山(富山県立大学　中川慎二)


## Disclaimer

   OPENFOAM® is a registered trade mark of OpenCFD Limited, the producer of the OpenFOAM software and owner of the OPENFOAM® and OpenCFD® trade marks. This offering is not approved or endorsed by OpenCFD Limited.


## 注意

　本資料の内容は，OpenFOAMユーザーガイド，プログラマーズガイド，OpenFOAM Wiki，CFD Online，その他多くの情報を参考にしています。開発者，情報発信者の皆様に深い謝意を表します。

　内容は，講師の個人的な経験（主に，卒研生等とのコードリーディング）から得た知識を共有するものです。この内容の正確性を保証することはできません。この情報を使用したことによって問題が生じた場合，その責任は負いかねますので，予めご了承ください。

## はじめに

この資料は，オープンCAE勉強会＠富山（第34回2015年7月25日）の「OpenFOAM 非ニュートンモデル カスタマイズ（秋山氏）」を補足する目的で作成されました。

講習会では，OpenFOAMのオリジナルコードを多く複製して，簡便にカスタムライブラリを作成する方法が説明されました。

この資料では，必要最小限のコードを複製して新たなライブラリを作成し，実行時にライブラリを指定する方法について解説します。

この資料を読むときは，前述の講習会資料も併せてご覧ください。

http://eddy.pu-toyama.ac.jp/%E3%82%AA%E3%83%BC%E3%83%97%E3%83%B3CAE%E5%8B%89%E5%BC%B7%E4%BC%9A-%E5%AF%8C%E5%B1%B1/?action=cabinet_action_main_download&block_id=99&room_id=1&cabinet_id=1&file_id=125&upload_id=255


## copy original files

ユーザのプロジェクトディレクトリに，srcディレクトリを作成する。

> mkdir $WM_PROJECT_USER_DIR/src

OpenFOAM標準のCrossPowerLawモデルのコードを，先ほどのディレクトリにコピーする。

> cp -r $WM_PROJECT_DIR/src/transportModels/incompressible/viscosityModels/CrossPowerLaw/ $WM_PROJECT_USER_DIR/src/CrossLaw/

コピーしたファイルの名前を，CrossPowerLawからCrossLawに変更する。CrossPowerLaw.depファイルが存在する場合には削除する。

> cd $WM_PROJECT_USER_DIR/src/CrossLaw/

> mv CrossPowerLaw.H CrossLaw.H

> mv CrossPowerLaw.C CrossLaw.C

> rm CrossPowerLaw.dep

同じディレクトリ内に，コンパイルに必要な情報をコピーする。非ニュートン粘性モデルライブラリのためのMakeディレクトリ（$WM_PROJECT_DIR/src/transportModels/incompressible/Make/）を，$WM_PROJECT_USER_DIR/src/CrossLaw/にコピーする。

> cp -R $WM_PROJECT_DIR/src/transportModels/incompressible/Make/ $WM_PROJECT_USER_DIR/src/CrossLaw/


## modify source files

> sed -i s/CrossPowerLaw/CrossLaw/g CrossLaw.H

> sed -i s/CrossPowerLaw/CrossLaw/g CrossLaw.C

Modify CrossLaw.H and CrossLaw.C as instructed by Mr.Akiyama

http://eddy.pu-toyama.ac.jp/%E3%82%AA%E3%83%BC%E3%83%97%E3%83%B3CAE%E5%8B%89%E5%BC%B7%E4%BC%9A-%E5%AF%8C%E5%B1%B1/#_99


    .C
        calcNu()  change equation
        comment out the lines about "nuInf_"
            lines 70, 100
    .H
        comment out the lines about "nuInf_"
            line 62


## Make ディレクトリの files と options ファイルを修正

新たなモデルを独立したライブラリとしてコンパイルするために，コンパイルに必要な情報を修正する。この情報は，$WM_PROJECT_USER_DIR/src/CrossLaw/Make ディレクトリ内に格納されている。

> cd $WM_PROJECT_USER_DIR/src/CrossLaw/Make

### Make/files

新しく作成したソースコード CrossLaw.C だけをコンパイルし，libCrossLawという名前のライブラリを作成する。filesファイルの内容を下記に書き換える。

```
CrossLaw.C

LIB = $(FOAM_USER_LIBBIN)/libCrossLaw
```

### Make/options

新しく作成したソースコード CrossLaw.C のコンパイルに必要なインクルードファイルの場所を指定する。また，CrossLawモデルを通常の非ニュートンモデルと入れ替えて使えるように，incompressibleTransportModelsライブラリを指定する。

```
EXE_INC = \
    -I$(LIB_SRC)/transportModels/incompressible/lnInclude \
    -I$(LIB_SRC)/finiteVolume/lnInclude

LIB_LIBS = \
    -lincompressibleTransportModels \
    -lfiniteVolume
```


## compile 

> cd $WM_PROJECT_USER_DIR/src/CrossLaw

> wmake

ライブラリなので，wmake libso としてもよい。wmake を実行するだけでも，自動的にでそうなっている。


## create a tutarial case

例題ケースを作成する。標準のoffsetCylinder 例題をコピーして，testCrossLawという名前のケースを作成する。

> cp -r  $FOAM_TUTORIALS/incompressible/nonNewtonianIcoFoam/offsetCylinder $WM_PROJECT_USER_DIR/run/tutorials/testCrossLaw/

testCrossLaw 例題ディレクトリに移動する。不要な情報を削除して，まっさらな状態に戻すため，foamCleanTutorials を実行する。

> cd $WM_PROJECT_USER_DIR/run/tutorials/testCrossLaw/

> foamCleanTutorials

作成したライブラリを使用するために，controlDictにライブラリを登録する。下記の内容を，controlDict最後に追加する。

> vi system/controlDict

```
libs
   (
        "libCrossLaw.so"
   );
```

CrossLawモデルを使うように，constant/transportPropertiesを変更する。元のファイルにおいて，CrossPowerLawをCrossLawに書き換える。

> sed -i s/CrossPowerLaw/CrossLaw/g ./constant/transportProperties

blockMesh でメッシュを生成，nonNewtonianIcoFoam を実行する。

> blockMesh
> nonNewtonianIcoFoam

----

付録

----

## ソルバからの書き出しにfunctionObjectが使えるのか？

秋山さん資料での，ソルバ修正箇所。

```
volScalarField strRatio
 (
 IOobject
 (
 "strRatio",
 runTime.timeName(),
 mesh,
 IOobject::NO_READ,
 IOobject::AUTO_WRITE
 ),
 mesh,
 dimensionedScalar("strRatio", dimensionSet(0,0,-1,0,0,0,0),
scalar(0.0))
 );
```

``` 
 strRatio = Foam::sqrt(2.0)*mag(symm(fvc::grad(U)));
 runTime.write();
```

OpenFOAMのバージョン2.0.0から追加された runtimeCodeCompilation 機能によって，ソルバを改造せずに，歪み速度を書き出すことを検討する。下記サイトの紹介では，簡単な使用例が記載されている。

http://www.openfoam.org/version2.0.0/runtime-control.php#runtimeCodeCompilation

```
functions 
( 
    pAverage 
    { 
        functionObjectLibs ("libutilityFunctionObjects.so"); 
        type coded; 
        redirectType average; 
        outputControl outputTime; 
        code 
        #{ 
            const volScalarField& p = 
                mesh().lookupObject<volScalarField>("p"); 
            Info<<"p avg:" << average(p) << endl;
        #}; 
    } 
); 
```

OpenFOAMユーザーガイドにも，説明がある。

OpenFOAM User Guide: 6.2 Function Objects
http://cfd.direct/openfoam/user-guide/function-objects/

ソルバ内に作成されている変数を書き出すためには，writeRegisteredObject タイプを使用する。変数を新たに定義して書き出すためには，これは使えない。少し込み入ったコードを作成する必要がある。

CFD-Online に，関連するディスカッションが存在する。下記のポスト#9を参考にして，サンプルを作成した。
http://www.cfd-online.com/Forums/openfoam-programming-development/99207-create-registered-object-runtime-using-functionobject.html#post384295

```
functions 
( 
    str 
    { 
        functionObjectLibs ("libutilityFunctionObjects.so"); 
        type coded; 
        redirectType strRate; // arbitral name
        outputControl outputTime; 
        code 
        #{ 
            const volVectorField& U = mesh().lookupObject<volVectorField>("U");
            static autoPtr<volScalarField> pField;

            if(!pField.valid())
            {
                Info << "Creating strRatio" << nl;
                pField.set
                (
                        new volScalarField
                        (
                             IOobject
                             (
                                 "strRatio",
                                  mesh().time().timeName(),
                                  U.mesh(),
                                  IOobject::NO_READ,
                                  IOobject::AUTO_WRITE
                             ),
                             Foam::sqrt(2.0)*mag(symm(fvc::grad(U)))
                        )
                );
            }

            volScalarField &strRatio = pField();

            strRatio.checkIn();

            strRatio = Foam::sqrt(2.0)*mag(symm(fvc::grad(U)));

            Info<<"strRate avg:" << average(strRatio) << endl; 

        #}; 
    } 
); 
```

function objects に関する参考サイト

function objects
http://www.geocities.co.jp/penguinitis2002/study/OpenFOAM/function_objects.html

 Tip Function Object writeRegisteredObject
 https://openfoamwiki.net/index.php/Tip_Function_Object_writeRegisteredObject
 
cfd-online
 http://www.cfd-online.com/Forums/openfoam/75049-how-get-density-field-compressible-flow.html#post254920
