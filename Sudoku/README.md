# Sudoku Clone

9×9 보드의 행, 열, 3×3 구역에 1부터 9까지의 숫자가 중복되지 않도록 채우는 Sudoku 규칙을 Unity와 C#으로 구현한 개인 학습 프로젝트입니다. 완성 보드를 생성한 뒤 숫자를 하나씩 제거하고 해답 수를 검사해, 최종 퍼즐이 하나의 해답만 갖도록 구성했습니다.

[WebGL로 플레이](https://jun-uni.github.io/Unity-Clone-Games/sudoku/)

## 플레이 화면

<img src="../sudoku.gif" width="794" alt="Sudoku 퍼즐 생성과 숫자 입력 플레이">

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 개발 형태 | 개인 프로젝트 |
| 개발 시기 | 2023.06 |
| 주요 구현 | 무작위 완성 보드, 유일 해답 퍼즐 생성, 빈칸 수 기반 50개 레벨, 셀 강조, 오답 제한, 레벨 해금 |
| 기술 | Unity, C#, 백트래킹, PlayerPrefs |
| 입력 | 마우스로 빈칸 선택, 숫자키 또는 숫자 키패드로 입력 |

## 구현 구조

먼저 후보 숫자를 무작위 순서로 대입하며 완성된 보드를 만듭니다. 이후 셀을 섞어 하나씩 비우고, 각 단계에서 해답이 정확히 하나인지 검사합니다. 해답이 여러 개가 되면 해당 숫자를 복원하고 다음 셀을 시도합니다. 완성 보드 생성과 해답 탐색은 왼쪽 위부터 행 우선 순서로 진행하며, 유일성 판단은 두 번째 해답을 찾는 순간 중단합니다.

1. 빈 9×9 보드에 백트래킹으로 완성 해답 생성
2. 셀 위치를 섞고 숫자를 하나씩 임시 제거
3. 해답 탐색을 최대 두 개까지만 진행
4. 해답이 하나일 때만 빈칸을 유지
5. 선택한 레벨의 빈칸 수에 도달하면 게임 보드 구성

## 핵심 코드

### 유일 해답을 유지하는 빈칸 생성

숫자를 하나 제거할 때마다 복사한 보드에서 해답 수를 검사합니다. 해답이 하나일 때만 제거 상태를 유지하고, 둘 이상이면 원래 숫자를 복원합니다.

```csharp
private static bool TryRemoveCells(int[][] board, int targetCount)
{
    if (targetCount == 0)
    {
        return true;
    }

    var positions = new List<int>(CellCount);
    for (var index = 0; index < CellCount; index++)
    {
        positions.Add(index);
    }

    Shuffle(positions);
    var removedCount = 0;

    foreach (var position in positions)
    {
        var row = position / Size;
        var column = position % Size;
        var originalValue = board[row][column];
        board[row][column] = 0;

        if (CountSolutions(CopyBoard(board), 0, 0, 2) == 1)
        {
            removedCount++;
            if (removedCount == targetCount)
            {
                return true;
            }
        }
        else
        {
            board[row][column] = originalValue;
        }
    }

    return removedCount == targetCount;
}
```

### 두 번째 해답에서 중단하는 행 우선 탐색

유일성 판단에는 전체 해답 수가 필요하지 않으므로 두 번째 해답을 찾는 순간 재귀 탐색을 끝냅니다. 빈칸은 원래 구현 방식대로 왼쪽 위부터 행 우선 순서로 탐색합니다.

```csharp
private static int CountSolutions(
    int[][] board,
    int row,
    int column,
    int limit)
{
    if (row == Size)
    {
        return 1;
    }

    var nextRow = column == Size - 1 ? row + 1 : row;
    var nextColumn = column == Size - 1 ? 0 : column + 1;

    if (board[row][column] != 0)
    {
        return CountSolutions(board, nextRow, nextColumn, limit);
    }

    var solutionCount = 0;

    for (var number = 1; number <= Size; number++)
    {
        if (!IsValid(board, row, column, number))
        {
            continue;
        }

        board[row][column] = number;
        solutionCount += CountSolutions(
            board,
            nextRow,
            nextColumn,
            limit - solutionCount);
        board[row][column] = 0;

        if (solutionCount >= limit)
        {
            return solutionCount;
        }
    }

    return solutionCount;
}
```

### 숫자 입력 경로 통합

상단 숫자키와 숫자 키패드를 하나의 순회에서 처리합니다. 입력된 숫자는 선택된 빈칸이 있을 때만 검증하며, 정답과 오답의 후속 처리를 분리했습니다.

```csharp
private static bool TryReadDigit(out int number)
{
    for (var index = 0; index < NumberRowKeys.Length; index++)
    {
        if (Input.GetKeyDown(NumberRowKeys[index]) || Input.GetKeyDown(KeypadKeys[index]))
        {
            number = index + 1;
            return true;
        }
    }

    number = 0;
    return false;
}

public void SubmitNumber(int number)
{
    var selectedCell = GameManager.Instance.CurrentInputBox;
    if (selectedCell == null || !sudoku.TryGetCoordinates(selectedCell, out var row, out var column))
    {
        return;
    }

    if (!string.IsNullOrEmpty(sudoku.GetCellText(row, column).text))
    {
        return;
    }

    if (sudoku.TryPlaceNumber(row, column, number))
    {
        GameManager.Instance.Correct();
        return;
    }

    GameManager.Instance.Wrong();
    UIManager.Instance.ShowWrongNumber(sudoku, row, column, number);
}
```

### 레벨 해금과 진행도 저장

각 레벨 번호는 퍼즐에서 비울 셀의 수로 사용합니다. 클리어한 최고 레벨만 저장하고, 다음 레벨까지 선택할 수 있도록 버튼 상태를 결정합니다.

```csharp
public void ClearLevel()
{
    var highestClearedLevel = PlayerPrefs.GetInt(ClearedLevelKey, 0);

    if (CurrentLevel > highestClearedLevel)
    {
        PlayerPrefs.SetInt(ClearedLevelKey, CurrentLevel);
        PlayerPrefs.Save();
    }

    GameOver(true);
}

private void CreateLevelButton(int level, int highestClearedLevel)
{
    var buttonObject = Instantiate(buttonPrefabs, transform, false);
    var button = buttonObject.GetComponent<Button>();
    button.GetComponentInChildren<TextMeshProUGUI>().text = level.ToString();
    button.interactable = level <= highestClearedLevel + 1;

    var selectedLevel = level;
    button.onClick.AddListener(() => OnButtonClick(selectedLevel));
}
```
