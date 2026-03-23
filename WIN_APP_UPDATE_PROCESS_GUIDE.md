# Windows 앱 업데이트 프로세스 가이드

Windows 데스크톱 앱(WPF, WinForms 등)에서 자동/수동 업데이트 기능을 구현할 때 참고하는 범용 가이드이다.
개발자와 AI 에이전트 모두를 대상으로 하며, 프로젝트에 맞게 적용한다.

---

## 1. 아키텍처 개요

### 1.1 계층 구조

업데이트 시스템은 세 계층으로 분리한다.

```
┌─────────────────────────────────────────────────────────────┐
│  View (Dialog/Window)                                       │
│  - 검색 진행 상태 표시 (단계별 아이콘)                           │
│  - 결과 표시 (버전 정보, 에러 메시지)                           │
│  - 사용자 액션 버튼 (업데이트/나중에/취소/닫기)                   │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │ 이벤트/바인딩
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  ViewModel                                                  │
│  - 업데이트 로직 제어 (CheckUpdateAsync, RunUpdateAsync)      │
│  - 다중 소스 병렬 검색                                        │
│  - 버전 비교                                                 │
│  - 상태 관리 (IsUpdateChecking, HasUpdate)                  │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │ 의존성 주입
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Service                                                    │
│  - GitHub API 통신 (릴리스 정보 조회, 다운로드)                 │
│  - 로컬/네트워크 경로 검색                                     │
│  - 토큰 관리                                                 │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 핵심 컴포넌트

| 컴포넌트 | 역할 | 예시 클래스명 |
|----------|------|--------------|
| **ViewModel** | 업데이트 로직 총괄, 상태 관리 | `MainViewModel` |
| **UpdateService** | 외부 소스(GitHub) 통신 | `GitHubUpdateService` |
| **UpdateDialog** | 통합 UI (검색/결과/액션) | `UpdateLoadingDialog` |

---

## 2. 다중 소스 병렬 검색

### 2.1 검색 소스 우선순위

업데이트 인스톨러는 여러 소스에서 **병렬로** 검색하되, 가장 먼저 성공한 결과를 사용한다.

| 순서 | 소스 | 용도 | 타임아웃 |
|------|------|------|----------|
| 0 | 로컬 배포 경로 | `{DEPLOY_ROOT}/{APP_NAME}` | 1초 |
| 1 | UNC 경로 | `\\{SERVER}\{SHARE}\{APP_NAME}` | 1초 |
| 2 | GitHub Releases | 인터넷 접근 가능 시 | 3초 |

### 2.2 병렬 검색 구현 패턴

```csharp
private async Task<InstallerInfo?> FindLatestInstallerAsync(CancellationToken ct = default)
{
    // 첫 번째 성공 시 나머지 태스크 취소
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    var linkedToken = linkedCts.Token;

    var sourceTasks = new (int StepIndex, Task<InstallerInfo?> Task)[]
    {
        (0, FindLocalInstallerAsync(@"{LOCAL_PATH}",  linkedToken)),
        (1, FindLocalInstallerAsync(@"{UNC_PATH}",    linkedToken)),
        (2, FindGitHubInstallerAsync(linkedToken)),
    };

    var pendingTasks = sourceTasks.Select(s => s.Task).ToList();
    InstallerInfo? firstResult = null;

    while (pendingTasks.Count > 0)
    {
        var completed = await Task.WhenAny(pendingTasks);
        var stepIndex = sourceTasks.First(s => s.Task == completed).StepIndex;
        pendingTasks.Remove(completed);

        InstallerInfo? result;
        try { result = await completed; }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            throw; // 사용자 취소: 상위로 전파
        }
        catch (OperationCanceledException)
        {
            continue; // 내부 취소: 무시
        }
        catch (Exception ex)
        {
            // 실패 이벤트 발화
            UpdateStepCompleted?.Invoke(this, (stepIndex, false));
            continue;
        }

        if (result is not null)
        {
            UpdateStepCompleted?.Invoke(this, (stepIndex, true));
            if (firstResult is null)
            {
                firstResult = result;
                linkedCts.Cancel(); // 나머지 태스크 취소
            }
        }
        else
        {
            UpdateStepCompleted?.Invoke(this, (stepIndex, false));
        }
    }

    return firstResult;
}
```

### 2.3 로컬 경로 검색 (타임아웃 적용)

네트워크 드라이브나 UNC 경로는 접근 불가 시 오래 걸릴 수 있으므로 타임아웃을 적용한다.

```csharp
private async Task<InstallerInfo?> FindLocalInstallerAsync(string sourceDir, CancellationToken ct)
{
    const int TimeoutMs = 1000; // 1초 타임아웃

    // Directory.Exists 체크 (타임아웃 적용)
    var existsTask = Task.Run(() =>
    {
        try { return Directory.Exists(sourceDir); }
        catch { return false; }
    });
    var timeoutTask = Task.Delay(TimeoutMs, ct);
    var completedTask = await Task.WhenAny(existsTask, timeoutTask);

    if (completedTask == timeoutTask || ct.IsCancellationRequested)
        return null;

    var exists = await existsTask;
    if (!exists) return null;

    // 파일 검색 (타임아웃 적용)
    var searchPattern = $"{INSTALLER_PREFIX}*.exe";
    var filesTask = Task.Run(() => Directory.GetFiles(sourceDir, searchPattern));
    timeoutTask = Task.Delay(TimeoutMs, ct);
    completedTask = await Task.WhenAny(filesTask, timeoutTask);

    if (completedTask == timeoutTask || ct.IsCancellationRequested)
        return null;

    var files = await filesTask;
    
    // 가장 높은 버전 찾기
    InstallerInfo? best = null;
    foreach (var file in files)
    {
        var fileName = Path.GetFileNameWithoutExtension(file);
        if (!fileName.StartsWith(INSTALLER_PREFIX)) continue;
        var version = fileName.Substring(INSTALLER_PREFIX.Length);
        if (string.IsNullOrEmpty(version)) continue;
        if (best is null || CompareVersions(version, best.Version) > 0)
            best = new InstallerInfo(version, LocalPath: file);
    }

    return best;
}
```

---

## 3. GitHub Releases 연동

### 3.1 서비스 클래스 구조

```csharp
public sealed class GitHubUpdateService
{
    private readonly HttpClient _httpClient;
    private readonly string? _token;
    private readonly DateTime? _tokenExpiry;

    private const string RepoOwner = "{GITHUB_ORG}";
    private const string RepoName = "{GITHUB_REPO}";
    private const string InstallerPrefix = "{APP_NAME}Install_";
    private const int ApiTimeoutSeconds = 3;
    private const int DownloadTimeoutSeconds = 300;

    public GitHubUpdateService(HttpClient httpClient, string? token = null, DateTime? tokenExpiry = null)
    {
        _httpClient = httpClient;
        _tokenExpiry = tokenExpiry;
        _token = string.IsNullOrWhiteSpace(token) ? FallbackToken : token;

        if (!string.IsNullOrWhiteSpace(_token))
        {
            _httpClient.DefaultRequestHeaders.Authorization =
                new AuthenticationHeaderValue("Bearer", _token);
        }
    }

    public bool IsTokenExpired() =>
        _tokenExpiry.HasValue && DateTime.Today > _tokenExpiry.Value.Date;
}
```

### 3.2 최신 릴리스 조회

```csharp
public async Task<(string Version, string DownloadUrl)?> GetLatestReleaseAsync(CancellationToken ct = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(ApiTimeoutSeconds));

    var url = $"https://api.github.com/repos/{RepoOwner}/{RepoName}/releases/latest";
    var response = await _httpClient.GetAsync(url, cts.Token);

    if (!response.IsSuccessStatusCode) return null;

    var release = await response.Content.ReadFromJsonAsync<GitHubRelease>(cancellationToken: cts.Token);
    if (release is null) return null;

    // 태그에서 버전 추출 (v1.2.3 → 1.2.3)
    var version = release.TagName?.TrimStart('v') ?? string.Empty;
    if (string.IsNullOrEmpty(version)) return null;

    // 인스톨러 에셋 찾기
    var installerAsset = release.Assets?.FirstOrDefault(a =>
        a.Name?.StartsWith(InstallerPrefix, StringComparison.OrdinalIgnoreCase) == true &&
        a.Name?.EndsWith(".exe", StringComparison.OrdinalIgnoreCase) == true);

    if (installerAsset is null || installerAsset.Id == 0) return null;

    // Private 리포지토리에서는 browser_download_url 대신 API URL을 사용해야 한다.
    // 자세한 이유는 "14. 트러블슈팅: Private 리포 에셋 다운로드" 참조.
    var apiDownloadUrl = $"https://api.github.com/repos/{RepoOwner}/{RepoName}/releases/assets/{installerAsset.Id}";

    return (version, apiDownloadUrl);
}
```

### 3.3 GitHub API 응답 모델

```csharp
private sealed class GitHubRelease
{
    [JsonPropertyName("tag_name")]
    public string? TagName { get; set; }

    [JsonPropertyName("name")]
    public string? Name { get; set; }

    [JsonPropertyName("assets")]
    public List<GitHubAsset>? Assets { get; set; }
}

private sealed class GitHubAsset
{
    [JsonPropertyName("id")]
    public long Id { get; set; }

    [JsonPropertyName("name")]
    public string? Name { get; set; }

    [JsonPropertyName("browser_download_url")]
    public string? BrowserDownloadUrl { get; set; }

    [JsonPropertyName("size")]
    public long Size { get; set; }
}
```

### 3.4 인스톨러 다운로드 (진행률 콜백)

```csharp
public async Task<string?> DownloadInstallerAsync(
    string downloadUrl,
    string version,
    Action<int>? progressCallback = null,
    CancellationToken ct = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(DownloadTimeoutSeconds));

    progressCallback?.Invoke(0);

    // API URL + Accept: application/octet-stream 으로 바이너리 다운로드
    // Private 리포에서는 이 헤더가 필수다. "14. 트러블슈팅" 참조.
    var request = new HttpRequestMessage(HttpMethod.Get, downloadUrl);
    request.Headers.Accept.Add(
        new System.Net.Http.Headers.MediaTypeWithQualityHeaderValue("application/octet-stream"));

    using var response = await _httpClient.SendAsync(
        request,
        HttpCompletionOption.ResponseHeadersRead,
        cts.Token);

    response.EnsureSuccessStatusCode();

    var totalBytes = response.Content.Headers.ContentLength ?? -1;
    var tempDir = Path.GetTempPath();
    var fileName = $"{InstallerPrefix}{version}.exe";
    var destPath = Path.Combine(tempDir, fileName);

    await using var contentStream = await response.Content.ReadAsStreamAsync(cts.Token);
    await using var fileStream = new FileStream(destPath, FileMode.Create, FileAccess.Write, FileShare.None, 81920, true);

    var buffer = new byte[81920];
    long totalRead = 0;
    int lastReportedProgress = 0;

    while (true)
    {
        var bytesRead = await contentStream.ReadAsync(buffer, cts.Token);
        if (bytesRead == 0) break;

        await fileStream.WriteAsync(buffer.AsMemory(0, bytesRead), cts.Token);
        totalRead += bytesRead;

        if (totalBytes > 0)
        {
            var progress = (int)(totalRead * 100 / totalBytes);
            if (progress > lastReportedProgress)
            {
                lastReportedProgress = progress;
                progressCallback?.Invoke(progress);
            }
        }
    }

    progressCallback?.Invoke(100);
    return destPath;
}
```

---

## 4. 버전 비교

### 4.1 버전 파일

앱 실행 경로에 `VERSION` 파일을 두고 현재 버전을 관리한다.

```
1.2.34
```

### 4.2 버전 읽기

```csharp
private static string GetCurrentVersion()
{
    try
    {
        var exeDir = Path.GetDirectoryName(Environment.ProcessPath) ?? AppContext.BaseDirectory;
        var versionFile = Path.Combine(exeDir, "VERSION");
        if (File.Exists(versionFile))
            return File.ReadAllText(versionFile).Trim();
    }
    catch { }
    return string.Empty;
}
```

### 4.3 버전 비교 함수

```csharp
/// <summary>
/// 버전 문자열을 비교한다.
/// a < b면 음수, a == b면 0, a > b면 양수.
/// </summary>
private static int CompareVersions(string a, string b)
{
    var partsA = a.Split('.').Select(s => int.TryParse(s, out var n) ? n : 0).ToArray();
    var partsB = b.Split('.').Select(s => int.TryParse(s, out var n) ? n : 0).ToArray();

    for (int i = 0; i < Math.Max(partsA.Length, partsB.Length); i++)
    {
        var va = i < partsA.Length ? partsA[i] : 0;
        var vb = i < partsB.Length ? partsB[i] : 0;
        if (va != vb) return va.CompareTo(vb);
    }
    return 0;
}
```

---

## 5. UI 패턴: 통합 다이얼로그

### 5.1 다이얼로그 모드

하나의 다이얼로그에서 여러 상태를 전환하여 표시한다.

```csharp
public enum UpdateDialogMode
{
    /// <summary>업데이트 검색 중 (3단계 진행 + 취소 버튼)</summary>
    Searching,

    /// <summary>업데이트 발견 (버전 정보 + 업데이트/나중에 버튼)</summary>
    UpdateFound,

    /// <summary>업데이트 없음 (에러 메시지 + 닫기 버튼)</summary>
    NoUpdate,

    /// <summary>이미 최신 버전 (메시지 + 닫기 버튼)</summary>
    AlreadyLatest,
}
```

### 5.2 단계별 상태

```csharp
public enum UpdateStepStatus
{
    Pending,   // 대기 중 (회색 원)
    Running,   // 진행 중 (스피너)
    Success,   // 성공 (체크 아이콘)
    Failed,    // 실패 (취소선)
}
```

### 5.3 모드 전환 메서드

```csharp
private void SetMode(UpdateDialogMode mode)
{
    _currentMode = mode;

    // 헤더 아이콘
    HeaderSpinner.Visibility     = mode == UpdateDialogMode.Searching ? Visibility.Visible : Visibility.Collapsed;
    HeaderSuccessIcon.Visibility = mode == UpdateDialogMode.UpdateFound ? Visibility.Visible : Visibility.Collapsed;
    HeaderInfoIcon.Visibility    = (mode == UpdateDialogMode.NoUpdate || mode == UpdateDialogMode.AlreadyLatest)
                                   ? Visibility.Visible : Visibility.Collapsed;

    // 헤더 텍스트
    HeaderText.Text = mode switch
    {
        UpdateDialogMode.Searching    => "업데이트 확인 중",
        UpdateDialogMode.UpdateFound  => "새 업데이트 발견",
        UpdateDialogMode.NoUpdate     => "업데이트 확인 실패",
        UpdateDialogMode.AlreadyLatest => "최신 버전 사용 중",
        _ => "업데이트 확인",
    };

    // 결과 패널
    ResultPanel.Visibility = (mode == UpdateDialogMode.Searching)
                             ? Visibility.Collapsed : Visibility.Visible;

    // 버튼 가시성
    CancelButton.Visibility = mode == UpdateDialogMode.Searching ? Visibility.Visible : Visibility.Collapsed;
    UpdateButton.Visibility = mode == UpdateDialogMode.UpdateFound ? Visibility.Visible : Visibility.Collapsed;
    LaterButton.Visibility  = mode == UpdateDialogMode.UpdateFound ? Visibility.Visible : Visibility.Collapsed;
    CloseButton.Visibility  = (mode == UpdateDialogMode.NoUpdate || mode == UpdateDialogMode.AlreadyLatest)
                              ? Visibility.Visible : Visibility.Collapsed;
}
```

### 5.4 단계 상태 업데이트

```csharp
public void SetStepStatus(int stepIndex, UpdateStepStatus status)
{
    var (pending, running, success, failed, text) = stepIndex switch
    {
        0 => (Step0Pending, Step0Running, Step0Success, Step0Failed, Step0Text),
        1 => (Step1Pending, Step1Running, Step1Success, Step1Failed, Step1Text),
        2 => (Step2Pending, Step2Running, Step2Success, Step2Failed, Step2Text),
        _ => throw new ArgumentOutOfRangeException(nameof(stepIndex)),
    };

    // 모두 숨김 후 해당 상태만 표시
    pending.Visibility  = Visibility.Collapsed;
    running.Visibility  = Visibility.Collapsed;
    success.Visibility  = Visibility.Collapsed;
    failed.Visibility   = Visibility.Collapsed;

    switch (status)
    {
        case UpdateStepStatus.Pending:
            pending.Visibility = Visibility.Visible;
            text.Foreground    = (Brush)FindResource("Text1Brush");
            text.TextDecorations = null;
            break;

        case UpdateStepStatus.Running:
            running.Visibility = Visibility.Visible;
            text.Foreground    = (Brush)FindResource("Text0Brush");
            text.TextDecorations = null;
            break;

        case UpdateStepStatus.Success:
            success.Visibility = Visibility.Visible;
            text.Foreground    = new SolidColorBrush(Color.FromRgb(0x4C, 0xAF, 0x50)); // 녹색
            text.TextDecorations = null;
            break;

        case UpdateStepStatus.Failed:
            failed.Visibility  = Visibility.Visible;
            text.Foreground    = (Brush)FindResource("Text2Brush");
            text.TextDecorations = TextDecorations.Strikethrough;
            break;
    }
}
```

---

## 6. 인스톨러 실행

### 6.1 네트워크 경로 보안 경고 방지

Windows는 네트워크 경로에서 직접 실행 시 보안 경고를 표시한다.
로컬 TEMP 폴더로 복사 후 실행하여 이를 방지한다.

```csharp
public void ExecuteUpdate(string installerPath)
{
    string localPath;

    // 이미 TEMP 경로에 있으면 복사 불필요 (GitHub 다운로드의 경우)
    if (installerPath.StartsWith(Path.GetTempPath(), StringComparison.OrdinalIgnoreCase))
    {
        localPath = installerPath;
    }
    else
    {
        // 네트워크 경로 → 로컬 TEMP 폴더로 복사
        localPath = CopyInstallerToTemp(installerPath);
    }

    Process.Start(new ProcessStartInfo
    {
        FileName        = localPath,
        Arguments       = "--quiet",
        UseShellExecute = true,
    });
}

private string CopyInstallerToTemp(string sourcePath)
{
    var tempDir = Path.GetTempPath();
    var fileName = Path.GetFileName(sourcePath);
    var destPath = Path.Combine(tempDir, fileName);

    File.Copy(sourcePath, destPath, overwrite: true);
    return destPath;
}
```

### 6.2 --quiet 모드

인스톨러는 `--quiet` 인자를 지원하여 사용자 확인 없이 자동 업데이트를 수행한다.

- 실행 중인 앱 프로세스 종료
- 파일 추출/교체
- 앱 재시작

---

## 7. ViewModel 이벤트 패턴

### 7.1 이벤트 정의

```csharp
// 단계 완료 시 발생 (StepIndex 0~2, Success 여부)
public event EventHandler<(int StepIndex, bool Success)>? UpdateStepCompleted;

// 업데이트 발견 시 발생 (현재 버전, 새 버전, 인스톨러 경로)
public event EventHandler<(string CurrentVersion, string NewVersion, string InstallerPath)>? UpdateConfirmRequested;

// 인스톨러를 찾을 수 없을 때 발생
public event EventHandler? UpdateNotFoundRequested;

// 이미 최신 버전일 때 발생
public event EventHandler? AlreadyLatestVersionRequested;

// 오류 발생 시 발생 (오류 메시지)
public event EventHandler<string>? UpdateErrorRequested;
```

### 7.2 View에서 이벤트 핸들링

```csharp
// MainWindow.xaml.cs

private void SubscribeUpdateEvents()
{
    _vm.UpdateStepCompleted += OnUpdateStepCompleted;
    _vm.UpdateConfirmRequested += OnUpdateConfirmRequested;
    _vm.UpdateNotFoundRequested += OnUpdateNotFoundRequested;
    _vm.AlreadyLatestVersionRequested += OnAlreadyLatestVersionRequested;
    _vm.UpdateErrorRequested += OnUpdateErrorRequested;
}

private void OnUpdateStepCompleted(object? sender, (int StepIndex, bool Success) e)
{
    _updateDialog?.SetStepStatus(e.StepIndex,
        e.Success ? UpdateStepStatus.Success : UpdateStepStatus.Failed);
}

private void OnUpdateConfirmRequested(object? sender, (string CurrentVersion, string NewVersion, string InstallerPath) e)
{
    _updateDialog?.ShowUpdateFound(e.CurrentVersion, e.NewVersion, e.InstallerPath);
}

private void OnUpdateNotFoundRequested(object? sender, EventArgs e)
{
    _updateDialog?.ShowNoUpdateFound();
}

private void OnAlreadyLatestVersionRequested(object? sender, EventArgs e)
{
    _updateDialog?.ShowAlreadyLatest(GetCurrentVersion());
}

private void OnUpdateErrorRequested(object? sender, string errorMessage)
{
    _updateDialog?.ShowError(errorMessage);
}
```

### 7.3 이벤트 흐름도

검색 시작부터 결과 표시까지의 전체 이벤트 흐름이다.

```
[RunUpdateAsync 시작]
  │
  ├─ IsUpdateChecking = true
  │   └─ MainWindow: 다이얼로그 생성, 3단계 모두 Running 상태 설정
  │
  ├─ FindLatestInstallerAsync() 병렬 실행
  │   ├─ Step 0 완료 → UpdateStepCompleted(0, success)
  │   ├─ Step 1 완료 → UpdateStepCompleted(1, success)
  │   └─ Step 2 완료 → UpdateStepCompleted(2, success)
  │   (첫 성공 시 linkedCts.Cancel()로 나머지 태스크 취소)
  │
  └─ 최종 결과 분기
       ├─ 인스톨러 발견 + 버전 높음 → UpdateConfirmRequested
       │   └─ Dialog: ShowUpdateFound() → UpdateFound 모드
       ├─ 인스톨러 발견 + 버전 같거나 낮음 → AlreadyLatestVersionRequested
       │   └─ Dialog: ShowAlreadyLatest() → AlreadyLatest 모드
       ├─ 인스톨러 못 찾음 → UpdateNotFoundRequested
       │   └─ Dialog: ShowNoUpdateFound() → NoUpdate 모드
       └─ 예외 발생 → UpdateErrorRequested
           └─ Dialog: ShowError() → NoUpdate 모드
```

**핵심 포인트:**

1. **병렬 실행, 순차 완료 이벤트**: 3개 소스가 동시에 실행되지만, 완료 이벤트는 각각 개별로 발화된다.
2. **첫 성공 시 취소**: 첫 번째 성공 결과가 나오면 `linkedCts.Cancel()`로 나머지 태스크를 취소하여 불필요한 대기를 방지한다.
3. **이벤트 기반 UI 업데이트**: ViewModel → View로 이벤트를 통해 상태를 전달하고, View에서 다이얼로그 메서드를 호출한다.

---

## 8. 앱 시작 시 자동 체크

### 8.1 백그라운드 체크 (HasUpdate 플래그만 설정)

```csharp
public MainViewModel(EnvironmentService envService, GitHubUpdateService gitHubService)
{
    _envService = envService;
    _gitHubService = gitHubService;
    _ = LoadDataAsync();
    _ = CheckUpdateAsync(); // 백그라운드에서 자동 체크
}

private async Task CheckUpdateAsync()
{
    await Task.Delay(2000); // 앱 로드 후 2초 대기

    var localVersion = GetCurrentVersion();
    if (string.IsNullOrEmpty(localVersion)) return;

    if (_gitHubService.IsTokenExpired()) return;

    var installer = await FindLatestInstallerAsync();
    if (installer is null) return;

    var compareResult = CompareVersions(localVersion, installer.Version);
    HasUpdate = compareResult < 0;
}
```

### 8.2 UI 표시

`HasUpdate` 플래그가 `true`일 때 업데이트 버튼에 뱃지를 표시한다.

```xml
<!-- 업데이트 버튼 (뱃지 포함) -->
<Button x:Name="UpdateButton" Command="{Binding RunUpdateCommand}">
    <Grid>
        <materialDesign:PackIcon Kind="Update"/>
        <!-- 업데이트 있을 때 느낌표 뱃지 -->
        <Ellipse Width="8" Height="8" Fill="Red"
                 Visibility="{Binding HasUpdate, Converter={StaticResource BoolToVisibilityConverter}}"
                 HorizontalAlignment="Right" VerticalAlignment="Top"/>
    </Grid>
</Button>
```

---

## 9. 인스톨러 파일명 규칙

| 항목 | 형식 | 예시 |
|------|------|------|
| 파일명 | `{APP_NAME}Install_{VERSION}.exe` | `MyAppInstall_1.2.34.exe` |
| 버전 형식 | `Major.Minor.Patch` | `1.2.34` |
| GitHub 태그 | `v{VERSION}` | `v1.2.34` |

---

## 10. 체크리스트

### 구현 시 확인 사항

- [ ] VERSION 파일이 앱과 함께 배포되는가?
- [ ] 로컬/UNC 경로 타임아웃이 적절한가? (1초 권장)
- [ ] GitHub API 타임아웃이 적절한가? (3초 권장)
- [ ] GitHub 토큰 만료일을 관리하고 있는가?
- [ ] 네트워크 경로 인스톨러 실행 시 TEMP 복사를 하는가?
- [ ] 인스톨러가 `--quiet` 모드를 지원하는가?
- [ ] 검색 실패 시 사용자에게 적절한 메시지를 표시하는가?

### GitHub 릴리스 설정

- [ ] 릴리스 태그가 `v{VERSION}` 형식인가?
- [ ] 인스톨러 에셋이 `{APP_NAME}Install_{VERSION}.exe` 형식인가?
- [ ] Private 리포의 경우 Contents 권한이 있는 PAT를 사용하는가?

---

## 11. 관련 문서

- [`APP_UPDATE_GUIDE.md`](APP_UPDATE_GUIDE.md) — PySide2 앱 업데이트 스플래시 스크린 구현 (별도 프로세스 방식)

---

## 12. GitHub PAT 인증 관리

### 12.1 PAT를 사용하는 이유

Private 리포지토리에서 릴리스 정보를 조회하거나, 인증되지 않은 요청의 API Rate Limit(시간당 60회)을 피하기 위해 GitHub PAT(Personal Access Token)를 사용한다.

### 12.2 PAT 설정 패턴

서비스 클래스 안에 PAT와 만료일을 상수/정적 멤버로 관리한다.

```csharp
public sealed class GitHubUpdateService : IDisposable
{
    // GitHub PAT 인증 — 만료일: {EXPIRY_DATE}
    private const string PATToken = "{YOUR_PAT}";
    private static readonly DateTime TokenExpiry = new DateTime({YEAR}, {MONTH}, {DAY});

    /// <summary>PAT 만료 여부를 반환한다.</summary>
    public static bool IsTokenExpired() => DateTime.Today > TokenExpiry;

    /// <summary>PAT 만료일 문자열 (사용자 안내 메시지용).</summary>
    public static string TokenExpiryString => "{EXPIRY_DATE}";

    private static HttpClient CreateDefaultClient()
    {
        var client = new HttpClient();
        client.DefaultRequestHeaders.UserAgent.ParseAdd("{APP_NAME}-Updater/1.0");
        // PAT가 만료되지 않은 경우에만 인증 헤더 추가
        if (!IsTokenExpired())
        {
            client.DefaultRequestHeaders.Authorization =
                new AuthenticationHeaderValue("Bearer", PATToken);
        }
        return client;
    }
}
```

### 12.3 PAT 만료 시 경고 다이얼로그 패턴

업데이트 버튼 클릭 시 PAT 만료 여부를 먼저 확인하고, 만료됐으면 사용자에게 알린 뒤 내부 네트워크 검색은 계속 진행한다.

```csharp
private void OnShowUpdateDialogRequested(object? sender, EventArgs e)
{
    // PAT 만료 여부 확인 — 만료됐어도 내부 네트워크 검색은 계속 진행한다
    if (GitHubUpdateService.IsTokenExpired())
    {
        MessageDialog.ShowWarning(
            $"GitHub 인증 토큰이 만료되었습니다 (만료일: {GitHubUpdateService.TokenExpiryString}).\n" +
            "인터넷을 통한 GitHub 업데이트 확인이 불가능합니다.\n" +
            "내부 네트워크 경로에서만 업데이트를 확인합니다.",
            "GitHub 토큰 만료");
    }

    // 이후 업데이트 다이얼로그 표시 및 검색 시작...
}
```

### 12.4 체크리스트

- [ ] PAT에 `Contents: Read` 권한이 있는가?
- [ ] PAT 만료일이 코드에 명시되어 있는가?
- [ ] 만료 시 사용자에게 안내 메시지를 표시하는가?
- [ ] 만료 후에도 내부 네트워크 검색은 계속 진행되는가?

---

## 13. 업데이트 다이얼로그 이동 방지

업데이트 진행 중 사용자가 다이얼로그나 메인 창을 이동시키지 못하도록 잠근다.

### 13.1 UpdateDialog 이동 불가 설정

`WindowStyle="None"`인 커스텀 다이얼로그는 타이틀바가 없으므로, 루트 Border의 `MouseDown` 핸들러에서 `DragMove()`를 호출하면 드래그 이동이 가능해진다.
업데이트 진행 중에는 이 핸들러에서 `DragMove()`를 호출하지 않아야 한다.

```csharp
// UpdateDialog.xaml.cs
private void RootBorder_MouseDown(object sender, MouseButtonEventArgs e)
{
    // 업데이트 진행 중 창 이동 방지 — DragMove() 호출 안 함
}
```

### 13.2 메인 윈도우 이동 방지 (업데이트 다이얼로그 표시 중)

표준 타이틀바가 있는 메인 윈도우는 WndProc으로 `SC_MOVE` 메시지를 차단한다.

```csharp
// MainWindow.xaml.cs
private bool _isUpdateDialogOpen;

// OnSourceInitialized에서 훅 등록
protected override void OnSourceInitialized(EventArgs e)
{
    base.OnSourceInitialized(e);
    var source = HwndSource.FromHwnd(new WindowInteropHelper(this).Handle);
    source?.AddHook(MainWindowWndProc);
}

// WndProc: 업데이트 다이얼로그가 열려있는 동안 SC_MOVE 차단
private IntPtr MainWindowWndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
{
    const int WM_SYSCOMMAND = 0x0112;
    const int SC_MOVE = 0xF010;
    if (_isUpdateDialogOpen && msg == WM_SYSCOMMAND && (wParam.ToInt32() & 0xFFF0) == SC_MOVE)
    {
        handled = true;
    }
    return IntPtr.Zero;
}

// 업데이트 다이얼로그 표시 시: 플래그 활성화
private void OnShowUpdateDialogRequested(object? sender, EventArgs e)
{
    _isUpdateDialogOpen = true;
    // ... 다이얼로그 생성 및 표시 ...
}

// 업데이트 다이얼로그 닫힘 시: 플래그 해제
private void OnUpdateDialogClosed(object? sender, EventArgs e)
{
    _isUpdateDialogOpen = false;
    // ... 이벤트 해제 ...
}
```

> ⚠️ **주의**: `System.Windows.Interop` 네임스페이스 using이 필요하다.

### 13.3 체크리스트

- [ ] UpdateDialog의 `RootBorder_MouseDown`에서 `DragMove()` 호출을 제거했는가?
- [ ] MainWindow에 `_isUpdateDialogOpen` 플래그가 있는가?
- [ ] `OnSourceInitialized`에서 WndProc 훅이 등록되는가?
- [ ] 다이얼로그 닫힘 시 `_isUpdateDialogOpen`을 `false`로 초기화하는가?

---

## 14. 트러블슈팅: Private 리포 에셋 다운로드

### 14.1 문제 현상

Private 리포지토리의 GitHub 릴리스에서 인스톨러를 다운로드할 때, `browser_download_url`을 사용하면 다운로드가 실패한다.
HTTP 응답은 성공(200)처럼 보이지만 실제로는 에러 XML이 반환되거나, 403 Forbidden이 발생한다.

### 14.2 원인

GitHub의 `browser_download_url`은 Amazon S3로 **302 리다이렉트**된다.
이때 `HttpClient`가 리다이렉트를 따라가면서 **`Authorization: Bearer {PAT}` 헤더도 함께 전달**한다.

S3는 자체 인증 체계를 사용하므로, 요청에 포함된 GitHub PAT를 **잘못된 인증 정보**로 간주하고 거부한다.

```
[요청 흐름 - 실패 케이스]

GET https://github.com/{owner}/{repo}/releases/download/v1.2.3/Installer.exe
Authorization: Bearer ghp_xxxx...
    |
    +-- 302 --> https://objects.githubusercontent.com/... (S3)
    |           Authorization: Bearer ghp_xxxx...  <-- 이 헤더가 문제
    |
    +-- S3 응답: 403 Forbidden 또는 에러 XML
       (GitHub 토큰을 S3 인증으로 해석하려다 실패)
```

### 14.3 해결 방법

`browser_download_url` 대신 **GitHub API URL**로 요청하고, `Accept: application/octet-stream` 헤더를 사용한다.

```
[요청 흐름 - 성공 케이스]

GET https://api.github.com/repos/{owner}/{repo}/releases/assets/{asset_id}
Authorization: Bearer ghp_xxxx...
Accept: application/octet-stream
    |
    +-- GitHub API가 인증 확인 후 302 --> S3 서명된 URL로 리다이렉트
    |   (Authorization 헤더 없이 리다이렉트 - S3 서명 URL 자체에 인증 포함)
    |
    +-- S3 응답: 200 OK + 바이너리 데이터
```

### 14.4 코드 수정 포인트

**1) 릴리스 조회 시 - API URL 생성**

```csharp
// 잘못된 방법: browser_download_url 직접 사용
return (version, installerAsset.BrowserDownloadUrl);

// 올바른 방법: API URL 생성 (asset id 사용)
var apiUrl = $"https://api.github.com/repos/{RepoOwner}/{RepoName}/releases/assets/{installerAsset.Id}";
return (version, apiUrl);
```

**2) 다운로드 시 - Accept 헤더 설정**

```csharp
// 잘못된 방법: 단순 GET 요청
using var response = await _httpClient.GetAsync(downloadUrl, ...);

// 올바른 방법: Accept: application/octet-stream 헤더 추가
var request = new HttpRequestMessage(HttpMethod.Get, downloadUrl);
request.Headers.Accept.Add(
    new MediaTypeWithQualityHeaderValue("application/octet-stream"));
using var response = await _httpClient.SendAsync(request, ...);
```

**3) 응답 모델 - Id 필드 추가**

```csharp
private sealed class GitHubAsset
{
    [JsonPropertyName("id")]
    public long Id { get; set; }     // API URL 생성에 필요

    [JsonPropertyName("name")]
    public string? Name { get; set; }

    // browser_download_url은 Private 리포에서 사용하면 안 되지만,
    // Public 리포 호환성을 위해 모델에는 유지한다.
    [JsonPropertyName("browser_download_url")]
    public string? BrowserDownloadUrl { get; set; }
}
```

### 14.5 Public vs Private 리포 동작 차이

| 항목 | Public 리포 | Private 리포 |
|------|------------|-------------|
| `browser_download_url` | 인증 없이 다운로드 가능 | S3 리다이렉트 시 Authorization 충돌로 실패 |
| API URL + `Accept: octet-stream` | 정상 동작 | 정상 동작 |
| PAT 필요 여부 | Rate Limit 회피용 (선택) | 필수 |

> **결론**: API URL + `Accept: application/octet-stream` 방식은 Public/Private 모두에서 동작하므로,
> 리포 공개 여부와 무관하게 이 방식을 기본으로 사용한다.

### 14.6 디버깅 팁

다운로드 실패 시 응답 본문을 확인하면 원인을 파악할 수 있다.

```csharp
if (!response.IsSuccessStatusCode)
{
    var body = await response.Content.ReadAsStringAsync();
    _logger.LogWarning("다운로드 실패: {StatusCode}, Body: {Body}",
        response.StatusCode, body);
    // S3 에러 시 XML 형식:
    // <Error><Code>InvalidArgument</Code><Message>...</Message></Error>
}
```
