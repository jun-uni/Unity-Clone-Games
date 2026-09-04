# 2048 Clone

같은 숫자의 타일을 이동 방향으로 밀고 합쳐 점수를 높이는 2048 규칙을 Unity와 C#으로 구현한 개인 학습 프로젝트입니다. 입력 방향과 관계없이 같은 이동 해석 절차를 사용하며, 한 번의 입력에서 생성된 타일이 다시 합쳐지지 않도록 병합 순서를 제어했습니다.

[WebGL로 플레이](https://jun-uni.github.io/Unity-Clone-Games/2048/)

## 플레이 화면

<img src="../2048.gif" width="900" alt="2048 타일 이동과 병합 플레이">

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 개발 형태 | 개인 프로젝트 |
| 개발 시기 | 2023.05 |
| 주요 구현 | 4×4 보드, 타일 이동과 병합, 점수와 최고 점수, 게임 오버, 재시작 |
| 기술 | Unity, C#, DOTween |
| 입력 | 방향키 또는 WASD, 게임 오버 후 R로 재시작 |

## 구현 구조

입력 처리는 이동 방향만 결정하고, 보드의 숫자 배열은 `BoardMoveResolver`가 해석합니다. 해석 결과에는 다음 보드 상태, 병합 위치, 획득 점수, 실제 이동 여부가 함께 담깁니다. 화면 갱신과 타일 애니메이션은 계산이 끝난 뒤 보드가 담당합니다.

1. 이동 방향의 끝을 선두로 각 행 또는 열 추출
2. 0을 제외해 타일 압축
3. 인접한 같은 숫자를 앞에서부터 한 차례만 병합
4. 결과를 원래 좌표계에 기록
5. 실제로 보드가 바뀐 경우에만 점수를 반영하고 새 타일 생성

## 핵심 코드

### 입력 이후의 처리 순서

입력, 규칙 계산, 화면 반영을 순서대로 분리했습니다. 움직일 수 없는 방향을 입력했을 때는 새 타일을 만들지 않습니다.

```csharp
private void TryMove(MoveDirection direction)
{
    var currentValues = board.CaptureValues();
    var result = BoardMoveResolver.Resolve(currentValues, direction);

    if (!result.Changed)
    {
        return;
    }

    board.Apply(result);
    GameManager.Instance.AddScore(result.ScoreGained);
    board.TryCreateBox();

    if (board.IsGameOver())
    {
        GameManager.Instance.EndGame();
    }
}
```

### 한 줄 압축과 병합

먼저 빈칸을 제거한 뒤 인접한 같은 숫자만 병합합니다. 병합이 일어나면 다음 원소를 건너뛰므로 `[2, 2, 2, 2]`는 `[4, 4, 0, 0]`이 됩니다.

```csharp
private static int[] ResolveLine(
    IReadOnlyList<int> source,
    out HashSet<int> mergedOffsets,
    out int scoreGained)
{
    var compacted = new List<int>(source.Count);

    foreach (var value in source)
    {
        if (value != 0)
        {
            compacted.Add(value);
        }
    }

    var result = new int[source.Count];
    mergedOffsets = new HashSet<int>();
    scoreGained = 0;
    var writeIndex = 0;

    for (var readIndex = 0; readIndex < compacted.Count; readIndex++)
    {
        var value = compacted[readIndex];

        if (readIndex + 1 < compacted.Count && value == compacted[readIndex + 1])
        {
            value *= 2;
            readIndex++;
            mergedOffsets.Add(writeIndex);
            scoreGained += value;
        }

        result[writeIndex++] = value;
    }

    return result;
}
```

### 네 방향을 하나의 규칙으로 변환

각 방향의 도착 지점부터 배열을 읽도록 좌표만 변환합니다. 압축과 병합 로직은 방향별로 반복하지 않고 동일한 함수를 사용합니다.

```csharp
private static (int row, int column) GetCoordinate(
    MoveDirection direction,
    int line,
    int offset,
    int size)
{
    return direction switch
    {
        MoveDirection.Left => (line, offset),
        MoveDirection.Right => (line, size - 1 - offset),
        MoveDirection.Down => (offset, line),
        MoveDirection.Up => (size - 1 - offset, line),
        _ => throw new ArgumentOutOfRangeException(nameof(direction), direction, null)
    };
}
```

### 게임 오버 판정

빈칸이 있으면 게임을 계속할 수 있습니다. 보드가 가득 찬 경우에는 오른쪽과 위쪽의 인접 타일만 비교해 모든 인접 쌍을 한 번씩 확인합니다.

```csharp
public bool IsGameOver()
{
    var values = CaptureValues();

    for (var row = 0; row < Size; row++)
    {
        for (var column = 0; column < Size; column++)
        {
            if (values[row, column] == 0)
            {
                return false;
            }

            if (column + 1 < Size && values[row, column] == values[row, column + 1])
            {
                return false;
            }

            if (row + 1 < Size && values[row, column] == values[row + 1, column])
            {
                return false;
            }
        }
    }

    return true;
}
```
