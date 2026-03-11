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

    if (installerAsset?.BrowserDownloadUrl is null) return null;

    return (version, installerAsset.BrowserDownloadUrl);
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

    using var response = await _httpClient.GetAsync(
        downloadUrl,
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
