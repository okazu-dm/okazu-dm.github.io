+++
date = "2018-05-28T00:00:48+09:00"
draft = false
title = "Goでディレクトリを列挙する"

+++

# 概要
Hugoを使ってみたテストとして日頃のメモを公開
goであるパスより下のディレクトリを列挙したい時にいくつかやり方がありそうだったという話


# 実装
とりあえず2種類はありそうだった

{{< highlight go >}}
package main

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Println("arg error. please specify path")
        os.Exit(1)
    }
    basepath := os.Args[1]
    fmt.Println("children ----")
    // findDirs(basepath, 2)
    walkFindDirs(basepath, 2)
}

func walkFindDirs(basepath string, depth int) {
    filepath.Walk(basepath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            if os.IsPermission(err) {
                return filepath.SkipDir
            }
            fmt.Printf("err: %v\n", err)
            return nil
        }
        ftype := "F"
        if info.IsDir() {
            ftype = "D"
            if info.Name() == ".git" {
                return filepath.SkipDir
            }
        }
        relpath, err := filepath.Rel(basepath, path)
        if err != nil {
            fmt.Printf("err: %v\n", err)
            return nil
        }
        if strings.Count(relpath, string(filepath.Separator)) == depth {
            // dig limited dpeth
            return filepath.SkipDir
        }
        fmt.Printf("%s: %s\n", ftype, path)
        return nil
    })

}

func findDirs(basepath wstring, depth int) {
    for currentDepth := 1; currentDepth <= depth; currentDepth++ {
        children, err := filepath.Glob(basepath + strings.Repeat("/*", currentDepth))
        if err != nil {
            fmt.Printf("err: %v\n", err)
            os.Exit(1)
        }
        for _, child := range children {
            fileinfo, err := os.Stat(child)
            if err != nil {
                fmt.Printf("err: %v\n", err)
                continue
            }
            if fileinfo.IsDir() {
                fmt.Println(child)
            }
        }
    }
}


{{< /highlight >}}

# 比較
walkFindDirsは `filepath.Walk()`, findDirsは`filepath.Glob()` を使っている。

深さを指定しやすいのはGlobを使う方だが、Walkは無視するディレクトリの指定がしやすい
(上のコードではGlobを使っている方の実装では特に無視するディレクトリを指定していない)

# (主にhugoの)感想
* go久々に書いてみるかと思って色々練習しているがまあ標準ライブラリで済む処理を書いている間は思想の統一感があるように感じて書きやすい気がする
* hugo、なんかテーマのテンプレがエラーを吐く気がするのでこの後よく見てみる(多分tomlのつもりでyamlを書いているとかそういう話)
* そもそもブログとかを書くタイプではないのでこれから使うかわからないけど、まあブログサービスに乗っからない場合はhugoはかなり選択肢の上位に来ると思った
    * 内容を書く以外の時間は、チュートリ読んでる時間が2割、テーマを選んでいる時間が3割、テンプレのエラー解決が5割(だいたい全部で45分くらい) => 楽
    * なんかテンプレの中でもgoのtemplateを使えるっぽさ
    * ブログサービスとかと同じでテーマは(おそらく)スパっと切り替えられる