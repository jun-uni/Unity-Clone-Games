# Suika Game Clone

과일을 떨어뜨리고 같은 종류끼리 합쳐 더 큰 과일로 진화시키는 규칙을 Unity와 C#으로 구현한 개인 학습 프로젝트입니다. Unity 2D 물리를 이용한 낙하와 충돌, 한 충돌에서 병합이 중복 실행되지 않도록 하는 상태 제어, 다음 과일과 점수 갱신 흐름을 구성했습니다.

[WebGL로 플레이](https://jun-uni.github.io/Unity-Clone-Games/suika/)

## 플레이 화면

<img src="../suika.gif" width="900" alt="Suika 과일 낙하와 병합 플레이">

## 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 개발 형태 | 개인 프로젝트 |
| 개발 시기 | 2024.01 |
| 주요 구현 | 과일 이동과 낙하, 동일 과일 병합, 11단계 진화, 다음 과일, 점수와 최고 점수, 게임 오버 |
| 기술 | Unity, C# |
| 입력 | A/D 또는 방향키로 이동, Space로 과일 떨어뜨리기 |

## 구현 구조

`SpawnerController`는 다음 과일을 선택하고 미리보기를 표시합니다. 입력을 받으면 미리보기와 별개의 과일을 생성해 물리 시뮬레이션에 넘깁니다. `Fruit`는 같은 단계의 과일과 충돌했을 때 양쪽을 병합 중 상태로 잠근 뒤 다음 단계 과일을 생성하고, `FruitCatalog`에서 점수와 프리팹을 조회합니다. 점수와 다음 과일 UI는 `GameManager`의 이벤트를 구독해 변경 시점에만 갱신합니다.

1. 생성 가능한 단계 중 다음 과일 선택
2. 물리 기능을 끈 별도 미리보기 인스턴스 표시
3. 입력 위치에 새 과일을 생성하고 낙하 시작
4. 같은 단계의 과일이 충돌하면 양쪽 병합 상태 잠금
5. 중간 위치에 다음 단계 과일 생성 후 점수 반영

## 핵심 코드

### 충돌당 한 번만 병합

충돌한 두 과일을 먼저 병합 중 상태로 전환하고 Collider를 비활성화합니다. 양쪽 오브젝트에서 충돌 콜백이 호출되더라도 먼저 실행된 처리만 병합을 진행합니다.

```csharp
private void OnCollisionEnter2D(Collision2D collision)
{
    if (isMerging || !collision.gameObject.TryGetComponent(out Fruit otherFruit))
    {
        return;
    }

    if (otherFruit.isMerging || otherFruit.fruitType != fruitType)
    {
        return;
    }

    MergeWith(otherFruit);
}

private void MergeWith(Fruit otherFruit)
{
    isMerging = true;
    otherFruit.isMerging = true;
    SetCollidersEnabled(false);
    otherFruit.SetCollidersEnabled(false);

    var midpoint = ((Vector2)transform.position
        + (Vector2)otherFruit.transform.position) * 0.5f;

    if (catalog.TryGetNext(fruitType, out var nextPrefab))
    {
        Instantiate(nextPrefab, midpoint, Quaternion.identity);
    }

    GameManager.Instance.AddScore(catalog.GetMergeScore(fruitType));
    Destroy(otherFruit.gameObject);
    Destroy(gameObject);
}
```

### 진화 데이터 중앙화

각 과일이 모든 단계의 프리팹을 직접 들고 있지 않도록, 단계별 프리팹과 병합 점수를 하나의 카탈로그에서 관리합니다. 현재 단계의 다음 인덱스를 조회해 진화 가능 여부도 함께 판단합니다.

```csharp
[CreateAssetMenu(menuName = "Suika/Fruit Catalog")]
public sealed class FruitCatalog : ScriptableObject
{
    [Serializable]
    private struct Entry
    {
        public Fruit prefab;
        public int mergeScore;
    }

    [SerializeField] private Entry[] entries;
    [SerializeField, Min(1)] private int spawnableTypeCount = 5;

    public bool TryGetNext(FruitType current, out Fruit nextPrefab)
    {
        var nextIndex = (int)current + 1;

        if (nextIndex >= entries.Length)
        {
            nextPrefab = null;
            return false;
        }

        nextPrefab = entries[nextIndex].prefab;
        return nextPrefab != null;
    }
}
```

### 미리보기와 실제 과일 분리

미리보기 인스턴스에는 Collider와 Rigidbody2D 시뮬레이션을 끄고, 낙하시킬 때는 원본 프리팹으로 새 인스턴스를 만듭니다. 미리보기용 상태가 실제 과일에 남지 않도록 생성 경로를 분리했습니다.

```csharp
private void DisplayNextFruit()
{
    if (previewFruit != null)
    {
        Destroy(previewFruit.gameObject);
    }

    selectedType = GameManager.Instance.TakeNextFruit(catalog);
    previewFruit = Instantiate(catalog.GetPrefab(selectedType), transform);
    previewFruit.transform.localPosition = previewOffset;
    previewFruit.SetPreviewMode();
}

private void SpawnSelectedFruit()
{
    var spawnedFruit = Instantiate(
        catalog.GetPrefab(selectedType),
        previewFruit.transform.position,
        Quaternion.identity);

    StartCoroutine(RestoreGameplayLayer(spawnedFruit.gameObject, 0.7f));
}
```

### 변경 시점에만 UI 갱신

점수와 다음 과일은 매 프레임 조회하지 않고 값이 바뀔 때 이벤트를 발행합니다. UI는 시작 시 현재 값을 한 번 표시하고 이후 변경 이벤트만 처리합니다.

```csharp
public void AddScore(int score)
{
    CurrentScore += score;

    if (CurrentScore > BestScore)
    {
        BestScore = CurrentScore;
    }

    ScoreChanged?.Invoke(CurrentScore, BestScore);
}

private void Start()
{
    GameManager.Instance.ScoreChanged += Refresh;
    Refresh(GameManager.Instance.CurrentScore, GameManager.Instance.BestScore);
}
```
