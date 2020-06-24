# Kubespray(kubespray)を既存のplaybookで利用する

1. [kubespray repo](https://github.com/kubernetes-sigs/kubespray)を個人/組織のgithubアカウントにフォークしてください:
   注意:
     * githubでフォークされたすべてのパブリックリポジトリもパブリックになるため、**機密データをパブリックフォークにコミットしないでください**。
     * フォークされたすべてのリポジトリのリストは、元のプロジェクトのgithubページから取得できます。

2. **forkリポジトリ** をサブモジュールとして既存のansibleリポジトリの目的のフォルダーに追加します(例: 3d/kubespray):
  ```git submodule add https://github.com/YOUR_GITHUB/kubespray.git kubespray```
  Gitは既存のansibleリポジトリに `.gitmodules`ファイルを作成します:

   ```ini
   [submodule "3d/kubespray"]
         path = 3d/kubespray
         url = https://github.com/YOUR_GITHUB/kubespray.git
   ```

3. サブモジュールのステータスを表示するようにgitを構成します:
```git config --global status.submoduleSummary true```

4. *元の* kubesprayリポジトリをupstreamとして追加します。
```git remote add upstream https://github.com/kubernetes-sigs/kubespray.git```

5. masterブランチをupstreamと同期する:

   ```ShellSession
      git checkout master
      git fetch upstream
      git merge upstream/master
      git push origin master
   ```

6. 作業環境で使用する新しいブランチを作成します:
```git checkout -b work```
    ***決して***コミットにリポジトリのmasterブランチを使用しないでください。

7. ansible.cfgファイルのライブラリとロールへのパスを変更します
 (ロールの名前は一意にする必要があります。kubesprayプロジェクトと同じ名前の既存のロールの名前を変更する必要がある場合があります):

   ```ini
   ...
   library       = 3d/kubespray/library/
   roles_path    = 3d/kubespray/roles/
   ...
   ```

8. kubesprayの `group_vars`フォルダから既存のプロジェクトの対応する` group_vars`フォルダに設定をコピーして変更します。
  *all.yml* 構成を別の名前(*kubespray.yml* など)に変更し、対応するグループをinventoryファイルに作成します。
  これには、kubernetesのセットアップに関連するすべてのホストグループが含まれます。

9. (存在する場合は)既存のグループのマッピングをkubesprayの命名に追加して、ansible inventoryファイルを変更します。
   例:

   ```ini
     ...
     #Kargo groups:
     [kube-node:children]
     kubenode

     [k8s-cluster:children]
     kubernetes

     [etcd:children]
     kubemaster
     kubemaster-ha

     [kube-master:children]
     kubemaster
     kubemaster-ha

     [kubespray:children]
     kubernetes
     ```

     * kubespray.yml設定ファイルを適用するために必要な最後のエントリーは、kubesprayプロジェクトのall.ymlから名前が変更されました。

10. cluster.ymlファイルを含めることで、既存のプレイブックにkubesprayタスクを含めることができるようになりました:

     ```yml
     - name: Include kubespray tasks
       include: 3d/kubespray/cluster.yml
     ```

     あるいは、cluster.yml から個別のタスクをansibleリポジトリにコピーすることもできます。

11. ansibleリポジトリに変更をコミットします。このサブモジュールフォルダは、フォークしたリポジトリのgitコミットハッシュへのリンクにすぎないことを覚えておいてください。
"work"ブランチを更新する際には、ansible レポにも変更をコミットする必要があります。
チームの他のメンバーは ````git submodule sync````, ````git submodule update --init```` を使って実際のコードを取得してください。

## コントリビュート

If you made useful changes or fixed a bug in existent kubespray repo, use this flow for PRs to original kubespray repo.
既存のkubesprayリポジトリに有用な変更を加えたり、バグを修正したりする場合、オリジナルのkubesprayリポジトリへのPRにはこのフローを利用してください。

1. [CNCF CLA](https://git.k8s.io/community/CLA.md)にサインする。

2. 作業ディレクトリをgitサブモジュールディレクトリに変更する(3d/kubespray)。

3. サブモジュールに必要なuser.nameとuser.emailを設定します。
もしkubesprayがあなたのリポジトリで1つのサブモジュールしかない場合は、次のようなものを使うことができます:
```git submodule foreach --recursive 'git config user.name "First Last" && git config user.email "your-email-addres@used.for.cncf"'```

4. upstreamのmasterと同期します:

   ```ShellSession
    git fetch upstream
    git merge upstream/master
    git push origin master
     ```

5. コントリビュートしたい特定の修正のための新しいブランチを作成する:
```git checkout -b fixes-name-date-index```
ブランチ名は日付やインデックスを追加することで、古いPRを追跡/削除するのに役立ちます。

6. "work"リポジトリでコミットのgitハッシュを見つけて、新しく作成した"fix"リポジトリに適用します:

     ```ShellSession
     git cherry-pick <COMMIT_HASH>
     ```

7. 複数の一時的なコミットがある場合は、[```git rebase -i```](https://eli.thegreenplace.net/2014/02/19/squashing-github-pull-requests-into-a-single-commit)を使ってつぶすようにしてください
また、インタラクティブなrebase (````git rebase -i HEAD~10````) を使って、オリジナルのリポジトリにコントリビュートしたくないコミットを削除することもできます。


8. 変更が入ったら、作業中に変更されている可能性があるので、もう一度upstreamのリポジトリを確認する必要があります。
正しいブランチにあることを確認してください:

```git status```
そして、upstreamからの変更をpullします(もしあれば):
```git pull --rebase upstream master```

9. これで、変更した内容を **fork** リポジトリに ```git push``` でプッシュします。
もしあなたのブランチが github に存在しない場合、git は ````git push -set-upstream origin fixes-name-date-index``` のような使い方を提案してきます。

10. ブラウザでフォークされたリポジトリを開くと、メインページに新しく作成したブランチのプルリクエストを作成するための提案が表示されます。
提案されているプルリクエストの差分を確認してください。
もし何か問題があれば ```git push origin --delete fixes-name-date-index```, ```git branch -D fixes-name-date-index``` を使ってgithub上の "fix" ブランチを安全に削除し、最初からすべての処理を行ってください。
問題がなければ、変更点の説明 (何をしているのか、なぜ必要なのか) を追加し、プルリクエストの作成を確認してください。
