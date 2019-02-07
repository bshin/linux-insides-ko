기여하기
================================================================================

[linux-insides](https://github.com/0xAX/linux-insides)에 기여하기를 원한다면, 하기의 간단한 rules을 따라주세요:

1. fork button을 눌러주세요:

    ![fork](http://oi58.tinypic.com/jj2trm.jpg)

2. 하기와 같이 자신의 account의 repository를 clone해 주세요:

    ```
    git clone git@github.com:your_github_username/linux-insides.git
    ```

3. 하기와 같이 새로운 branch를 만들어 주세요:

    ```
    git checkout -b "linux-bootstrap-1-fix"
    ```
    원하는 이름을 붙이시면 됩니다.

4. 변경해 주세요.

5. `contributors.md`에 자신을 추가하는 것을 잊지 마세요.

6. 변경점을 commit하고 push하고, Github에서 pull request 생성해 주세요.

**중요**

자신의 fork를 update하는 것을 잊지 마세요. 자신의 변경점을 만드는 동안, 다른 pull requests의 merge로 인해 `master` branch가 변경되어 conflicts를 생성할 수도 있습니다. 이것이 자신의 변경점을 push할 때마다 rebase하고 branch에 `master`와 conflicts가 없는지 확인해야 하는 이유입입니다.

감사합니다.
