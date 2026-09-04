# Wordle Clone

제한된 횟수 안에 정답을 추측하고, 각 문자의 포함 여부와 위치를 단서로 다음 입력을 결정하는 Wordle 규칙을 Unity와 C#으로 구현한 개인 학습 프로젝트입니다. 영어 단어와 한글 단어를 모두 지원하며, 중복 문자가 포함된 입력도 정답에 남아 있는 문자 수를 기준으로 판정합니다.

[WebGL로 플레이](https://jun-uni.github.io/Unity-Clone-Games/wordle/)

## 플레이 화면

### 한글 모드

<img src="../wordle-ko.gif" width="900" alt="Wordle 한글 모드 플레이">

### 영어 모드

<img src="../wordle-en.gif" width="900" alt="Wordle 영어 모드 플레이">

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 개발 형태 | 개인 프로젝트 |
| 개발 시기 | 2023.05 |
| 주요 구현 | 영어 5글자와 한글 5키 입력, 입력 안내 팝업, 6회 시도, 중복 문자 판정, 사전 검사, 키보드 상태 표시, 재시작 |
| 기술 | Unity, C#, HashSet |
| 입력 | 키보드 또는 화면 버튼, Enter로 제출, Backspace로 삭제, 게임 종료 후 R로 재시작 |

## 구현 구조

입력 계층은 영문 키를 게임에서 사용하는 문자로 변환해 현재 줄에 전달합니다. 다섯 칸이 채워지면 캐싱된 사전에서 단어를 완전 일치로 확인하고, 판정기는 정답과 입력만 받아 각 문자의 상태를 계산합니다. 게임 진행부는 판정 결과를 보드와 화면 키보드에 전달하고 정답 여부 또는 남은 시도 횟수에 따라 다음 줄이나 결과 화면으로 전환합니다.

한글 모드를 선택하면 초성·중성·종성을 각각 한 칸에 입력한다는 안내와 `가을 → ㄱ ㅏ ㅇ ㅡ ㄹ`, `장고 → ㅈ ㅏ ㅇ ㄱ ㅗ` 예시를 표시합니다. 결과 화면은 별도 배경 패널 위에 표시하며, 재시작해도 선택한 언어 모드를 유지합니다.

1. 키보드 또는 화면 버튼 입력을 현재 줄에 기록
2. 다섯 칸이 채워지면 사전에서 정확한 단어인지 확인
3. 정확한 위치를 먼저 확정한 뒤 남은 문자 수 계산
4. 다른 위치에 존재하는 문자와 없는 문자 판정
5. 타일과 화면 키보드에 결과를 반영하고 다음 시도 진행

## 핵심 코드

### 중복 문자를 고려한 두 단계 판정

첫 번째 순회에서 위치까지 일치하는 문자를 확정하고, 사용되지 않은 정답 문자만 개수로 기록합니다. 두 번째 순회에서는 남은 개수를 차감하며 다른 위치의 문자를 판정해 하나의 정답 문자가 여러 번 사용되지 않도록 했습니다.

```csharp
public static LetterState[] Evaluate(string answer, string guess)
{
    var result = new LetterState[answer.Length];
    var remainingLetters = new Dictionary<char, int>();

    for (var index = 0; index < answer.Length; index++)
    {
        if (answer[index] == guess[index])
        {
            result[index] = LetterState.Correct;
            continue;
        }

        remainingLetters.TryGetValue(answer[index], out var count);
        remainingLetters[answer[index]] = count + 1;
    }

    for (var index = 0; index < guess.Length; index++)
    {
        if (result[index] == LetterState.Correct)
        {
            continue;
        }

        var letter = guess[index];
        if (!remainingLetters.TryGetValue(letter, out var count) || count == 0)
        {
            result[index] = LetterState.Absent;
            continue;
        }

        result[index] = LetterState.Present;
        remainingLetters[letter] = count - 1;
    }

    return result;
}
```

### 사전 캐싱과 완전 일치 검사

게임 시작 시 영어와 한글 사전을 각각 `HashSet`에 한 번만 적재합니다. 제출할 때 파일을 다시 읽거나 부분 문자열을 검색하지 않고 입력한 단어 전체가 사전에 있는지 확인합니다.

```csharp
private readonly HashSet<string> englishDictionary =
    new(StringComparer.OrdinalIgnoreCase);
private readonly HashSet<string> koreanDictionary =
    new(StringComparer.Ordinal);

public bool IsWordValid(string guess, bool korean)
{
    return korean
        ? koreanDictionary.Contains(guess)
        : englishDictionary.Contains(guess);
}

private void LoadKoreanWords(TextAsset asset)
{
    foreach (var displayWord in ReadLines(asset))
    {
        var keys = KoreanTool.DecomposeToKeystrokes(displayWord);
        if (keys.Length != WordLength || !KoreanTool.IsSupportedKeySequence(keys))
        {
            continue;
        }

        if (koreanDictionary.Add(keys))
        {
            koreanWords.Add(new WordEntry(keys, displayWord));
        }
    }
}
```

### 한글을 동일한 5칸 규칙으로 변환

한글 모드는 완성된 음절을 두벌식 키 입력 단위로 분해합니다. 겹초성, 복합 모음과 겹받침을 기본 자모로 펼쳐 영어 모드와 같은 다섯 칸 판정 흐름을 사용합니다.

```csharp
public static string DecomposeToKeystrokes(string word)
{
    var result = new StringBuilder();

    foreach (var character in word)
    {
        if (character < HangulBase || character > HangulEnd)
        {
            if (KoreanLayout.IndexOf(character) >= 0)
            {
                result.Append(character);
            }

            continue;
        }

        var syllableIndex = character - HangulBase;
        var choseongIndex = syllableIndex / (JungCount * JongCount);
        var jungseongIndex = syllableIndex % (JungCount * JongCount) / JongCount;
        var jongseongIndex = syllableIndex % JongCount;

        result.Append(ExpandInitial(Chosung[choseongIndex]));
        result.Append(ExpandVowel(Jungsung[jungseongIndex]));

        if (jongseongIndex > 0)
        {
            result.Append(ExpandFinal(Jongsung[jongseongIndex]));
        }
    }

    return result.ToString();
}
```

### 제출과 시도 종료 흐름

현재 줄의 길이와 사전 포함 여부를 먼저 검사합니다. 유효한 입력만 판정해 타일과 화면 키보드에 전달하며, 정답 또는 마지막 시도에서 게임을 종료합니다.

```csharp
public void SubmitGuess()
{
    var currentLine = line[currentLineIndex];
    if (!currentLine.IsFull)
    {
        MessageRequested?.Invoke(GameMessage.NotEnoughLetters);
        return;
    }

    var guess = currentLine.GetWord();
    if (!checker.IsWordValid(guess, IsKorean))
    {
        MessageRequested?.Invoke(GameMessage.NotInWordList);
        return;
    }

    var states = checker.Evaluate(currentAnswer.Keys, guess);
    currentLine.ApplyStates(states);
    GuessEvaluated?.Invoke(guess, states);

    var isCorrect = Array.TrueForAll(states, state => state == LetterState.Correct);
    if (isCorrect || currentLineIndex == line.Length - 1)
    {
        IsGameOver = true;
        GameEnded?.Invoke(isCorrect, currentAnswer.Display);
        return;
    }

    currentLineIndex++;
}
```
