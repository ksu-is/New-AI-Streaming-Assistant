# New-Streamer-AI-assistant
@@ -0,0 +1,259 @@
# Plan: StreamHealthMonitor - Standalone Stream Health Monitoring Tool

## TL;DR
Create a standalone Python-based stream health monitoring tool that tracks OBS stream quality, system resources, network metrics, and platform analytics. Can optionally integrate with StreamerAI via shared SQLite database for coordinated alerts and control (e.g., pause AI during stream issues). Designed for reusability by any streamer, not just StreamerAI users.

## Architecture Approach

**Recommended**: Microservice architecture with three deployment modes:
1. **Standalone Mode**: Runs independently, monitors OBS + system metrics only
2. **Platform Integration Mode**: Adds Douyin/Bili viewer/chat metrics
3. **StreamerAI Integration Mode**: Full integration with StreamerAI via shared database

This maximizes reusability while enabling optional deep integration.

## Steps

**Phase 1: Core Monitoring Infrastructure** (*Foundation - all subsequent phases depend on this*)

1. Initialize new repository `StreamHealthMonitor` with Python 3.10+ project structure
   - Use Poetry for dependency management (consistent with StreamerAI)
   - Create modular package: `stream_monitor/` with submodules: `collectors/`, `storage/`, `alerting/`, `analysis/`
   - Add configuration via `config.yaml` + environment variables pattern

2. Implement SQLite storage layer (*parallel with step 3*)
   - Create schema with tables: `health_metrics`, `stream_alerts`, `stream_sessions`
   - Use Peewee ORM (same as StreamerAI for compatibility)
   - Add time-series data aggregation (5-sec, 1-min, 5-min buckets)
   - Implement auto-cleanup for metrics older than 24 hours

3. Build OBS WebSocket collector (*parallel with step 2*)
   - Integrate `obsws-python` library for OBS Studio 28+ WebSocket v5
   - Implement async polling for: FPS, bitrate, dropped frames, buffer fill, CPU/memory
   - Add connection retry logic with exponential backoff
   - Poll interval: 2 seconds for real-time metrics

4. Build system metrics collector
   - Use `psutil` to track: CPU (per-core), memory, disk I/O, network interfaces
   - Track OBS process specifically (if running) for detailed process metrics
   - Poll interval: 5 seconds for system-wide metrics

5. Create metrics aggregation engine
   - Collect data from all collectors into unified `StreamMetrics` dataclass
   - Calculate derived metrics: dropped frame percentage, bitrate variance, trend analysis
   - Store to SQLite with timestamp indexing

**Phase 2: Alerting & Analysis** (*depends on Phase 1 completion*)

6. Implement threshold-based alerting system
   - Define configurable thresholds for: dropped frames >5%, CPU >85%, bitrate variance >15%, network latency >200ms
   - Support three severity levels: INFO, WARNING, CRITICAL
   - Write alerts to `stream_alerts` table with duration tracking
   - Add alert suppression (don't re-alert on same issue within 5 minutes)

7. Build notification dispatcher
   - Support multiple notification channels: console logs, file logs, webhook (Discord/Slack)
   - Make notifications pluggable via abstract `NotificationHandler` interface
   - Add configuration for which alerts trigger which channels

8. Add trend analysis module
   - Calculate moving averages (1-min, 5-min, 15-min windows)
   - Detect anomalies: sudden bitrate drops, FPS degradation, sustained high CPU
   - Predict issues before they become critical (e.g., gradual memory leak detection)

**Phase 3: Platform Integration** (*depends on Phase 1, parallel with Phase 2*)

9. Create platform metrics collectors (optional feature)
   - **Douyin collector**: Use Playwright automation pattern from StreamerAI's `douyin.py` to extract viewer count, connection status
   - **Bili collector**: Use `blivedm` library from StreamerAI's `bili.py` for viewer metrics
   - Poll interval: 30 seconds for platform metrics
   - Make these optional dependencies (install via `poetry install --extras platform`)

10. Add chat analytics
    - Track chat message rate (messages/second)
    - Detect chat spam or unusual activity patterns
    - Correlate chat rate with stream health (e.g., chat drops when stream lags)

**Phase 4: StreamerAI Integration** (*depends on Phase 1 & 2*)

11. Implement StreamerAI database bridge
    - Add optional connection to StreamerAI's SQLite database at `StreamerAI/data/test.db`
    - Read StreamerAI's `Stream` and `Comment` tables to track AI activity metrics
    - Write monitoring data to shared location or separate schema
    - Use SQLite WAL mode for concurrent access (already enabled in StreamerAI)

12. Create control API for StreamerAI coordination
    - Expose simple SQLite-based control table: `monitor_commands` with fields: `command_type`, `status`, `timestamp`
    - Support commands: `PAUSE_AI`, `RESUME_AI`, `SKIP_PRODUCT`, `EMERGENCY_STOP`
    - StreamerAI polls this table in its main loop (similar to comment polling pattern)

13. Add AI-specific metrics tracking
    - Monitor TTS response times from StreamerAI's logs or database
    - Track GPT processing latency
    - Correlate AI performance with stream health (e.g., TTS slow → network issue?)

**Phase 5: User Interface & Visualization** (*depends on Phase 1 & 2*)

14. Build CLI dashboard
    - Use `rich` library for terminal UI with live-updating metrics display
    - Show real-time: FPS, bitrate, CPU, memory, alerts in formatted tables
    - Add command mode for: start/stop monitoring, export reports, configure thresholds

15. Create web dashboard (optional, future enhancement)
    - Use FastAPI + WebSockets for real-time updates
    - Build simple HTML/JS frontend with charts (Chart.js or Plotly)
    - Display historical metrics, trend graphs, alert timeline

**Phase 6: Testing & Documentation** (*parallel with all phases*)

16. Write comprehensive tests
    - Unit tests for each collector (mock OBS WebSocket, psutil responses)
    - Integration tests for database operations
    - End-to-end test with simulated stream session

17. Create documentation
    - README with installation, quick start, configuration guide
    - Architecture diagram showing components
    - Integration guide for StreamerAI users
    - Standalone usage guide for non-StreamerAI streamers

18. Package and release
    - Configure `pyproject.toml` with entry points: `stream-monitor`, `stream-monitor-dashboard`
    - Create GitHub Actions CI/CD for automated testing
    - Publish to PyPI for easy installation: `pip install stream-health-monitor`

## Relevant Files & Patterns

**From StreamerAI (for reference, not modification)**:
- [StreamerAI/streaming/main.py](StreamerAI/streaming/main.py) - subprocess management pattern, main loop structure, database polling
- [StreamerAI/database/database.py](StreamerAI/database/database.py) - Peewee ORM usage, SQLite WAL mode configuration
- [StreamerAI/streamchat/streamChatHandler.py](StreamerAI/streamchat/streamChatHandler.py) - rate limiting implementation, event handler pattern
- [StreamerAI/streamchat/douyin/douyin.py](StreamerAI/streamchat/douyin/douyin.py) - Playwright automation for Douyin
- [StreamerAI/streamchat/bili.py](StreamerAI/streamchat/bili.py) - `blivedm` async integration for Bilibili
- [StreamerAI/gpt/chains.py](StreamerAI/gpt/chains.py) - retry decorator pattern for API calls
- [StreamerAI/settings.py](StreamerAI/settings.py) - environment variable configuration pattern
- [pyproject.toml](pyproject.toml) - Poetry configuration with entry points

**New Repository Structure**:
```
StreamHealthMonitor/
├── pyproject.toml
├── README.md
├── config.yaml.example
├── stream_monitor/
│   ├── __init__.py
│   ├── main.py                    # CLI entry point
│   ├── config.py                  # Configuration management
│   ├── collectors/
│   │   ├── __init__.py
│   │   ├── base.py                # Abstract collector interface
│   │   ├── obs.py                 # OBS WebSocket collector
│   │   ├── system.py              # psutil system metrics
│   │   ├── douyin.py              # Optional: Douyin platform metrics
│   │   └── bili.py                # Optional: Bili platform metrics
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── database.py            # Peewee models & storage
│   │   └── aggregator.py          # Time-series aggregation
│   ├── alerting/
│   │   ├── __init__.py
│   │   ├── thresholds.py          # Threshold checking
│   │   ├── notifiers.py           # Notification handlers
│   │   └── analysis.py            # Trend analysis
│   ├── integration/
│   │   ├── __init__.py
│   │   └── streamer_ai.py         # StreamerAI bridge
│   └── ui/
│       ├── __init__.py
│       ├── cli_dashboard.py       # Rich terminal UI
│       └── web_dashboard.py       # Optional FastAPI dashboard
├── tests/
│   ├── test_collectors.py
│   ├── test_storage.py
│   └── test_alerting.py
└── docs/
    ├── architecture.md
    ├── integration_guide.md
    └── configuration.md
```

## Verification

**Phase 1 Verification**:
1. Run `stream-monitor --standalone` and verify it collects OBS metrics without errors
2. Check SQLite database contains populated `health_metrics` table with 2-second intervals
3. Simulate OBS disconnection and verify reconnection logic with exponential backoff
4. Verify psutil collects CPU/memory metrics for OBS process and system-wide

**Phase 2 Verification**:
1. Set dropped frames >5% threshold, simulate drops in OBS, verify alert fires in `stream_alerts` table
2. Test alert suppression: verify same alert doesn't fire twice within 5 minutes
3. Send webhook notification to Discord test channel, verify message formatting
4. Check trend analysis detects gradual CPU increase over 5-minute window

**Phase 3 Verification**:
1. Install platform extras: `poetry install --extras platform`
2. Run with Douyin integration, verify viewer count updates every 30 seconds
3. Test Bili chat rate tracking, correlate with actual chat activity

**Phase 4 Verification**:
1. Run StreamerAI and StreamHealthMonitor simultaneously pointing to same database
2. Trigger high latency alert in monitor, verify `PAUSE_AI` command written to control table
3. Verify StreamerAI pauses comment processing when command detected
4. Check AI metrics tracking: TTS latency stored correctly in monitoring database

**Phase 5 Verification**:
1. Launch `stream-monitor-dashboard` CLI, verify real-time metrics display updates every 2 seconds
2. Test CLI commands: `thresholds set cpu 90`, `export report --last 1h`
3. (If web dashboard built) Open localhost:8000, verify live charts update via WebSocket

**Phase 6 Verification**:
1. Run test suite: `poetry run pytest`, verify >90% code coverage
2. Build documentation site: `mkdocs build`, verify all pages render
3. Install from PyPI in clean environment: `pip install stream-health-monitor`, run help command
4. Test StreamerAI integration guide: follow docs step-by-step in fresh setup

## Decisions

**Technology Stack**:
- **Python 3.10+**: Matches StreamerAI for consistency
- **Poetry**: Dependency management, same as StreamerAI
- **SQLite + Peewee**: Compatible with StreamerAI's database layer, enables shared storage
- **asyncio**: Native async for non-blocking collectors
- **obsws-python**: Most maintained OBS WebSocket v5 library
- **psutil**: Cross-platform system metrics
- **rich**: Beautiful CLI dashboards without heavy dependencies

**Architecture Decision: Why Separate Repository**:
- Different release cycles (monitoring updates don't require AI changes)
- Reusable by non-AI streamers
- Independent dependency management (avoid bloating StreamerAI)
- Can run on different machines (e.g., monitor from laptop, AI on desktop)
- Easier testing in isolation

**Integration Decision: SQLite-based IPC vs HTTP API**:
- **Chosen**: SQLite shared database with WAL mode
- **Why**: StreamerAI already uses this pattern for subprocess communication, zero new dependencies, works offline, simple to debug
- **Alternative**: HTTP API would require running web server, adds network layer complexity

**Deployment Modes**:
- **Mode 1 (Standalone)**: Core deps only, monitors OBS + system
- **Mode 2 (Platform)**: Add `--extras platform` for Douyin/Bili
- **Mode 3 (StreamerAI)**: Add `--extras integration` for full AI coordination

## Further Considerations

1. **Should we support multiple OBS instances?** (e.g., streaming to multiple platforms simultaneously)
   - **Recommendation**: Start with single OBS, add multi-instance in v2.0 if requested
   
2. **Cloud storage for historical metrics?** Beyond local 24-hour retention
   - **Recommendation**: Phase 1 uses local SQLite only. Add optional cloud export (S3, PostgreSQL) in future version via plugin system
   
3. **Real-time vs. batch aggregation trade-off**
   - **Recommendation**: Hybrid approach - store raw metrics every 2 seconds for recent data (last hour), aggregate to 1-min buckets for older data to save space

4. **Alert fatigue prevention**: Too many alerts can be overwhelming
   - **Recommendation**: Implement alert grouping (single "Stream degraded" alert instead of 5 separate CPU/FPS/bitrate alerts) and configurable quiet periods

5. **GPU metrics**: NVIDIA-specific vs. cross-platform
   - **Recommendation**: Phase 1 uses psutil's basic GPU support. Add optional NVIDIA-specific metrics (temperature, VRAM) via `py3nvml` library in Phase 2
   - """Threshold checking for alerts"""
import logging
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Any, Dict, List, Optional, Tuple

from ..config import get_config, ThresholdConfig

logger = logging.getLogger(__name__)


class Severity(Enum):
    """Alert severity levels"""
    INFO = "INFO"
    WARNING = "WARNING"
    CRITICAL = "CRITICAL"


@dataclass
class Alert:
    """Represents an alert"""
    severity: Severity
    alert_type: str
    message: str
    value: float
    threshold: float
    source: str


class ThresholdChecker:
    """Check metrics against configured thresholds"""
    
    def __init__(self):
        self._config = get_config()
        self._thresholds = self._config.alerts.thresholds
        self._last_alert_time: Dict[str, datetime] = {}
    
    def check_all(
        self, 
        obs_metrics: Dict[str, Any], 
        system_metrics: Dict[str, Any]
    ) -> List[Alert]:
        """Check all metrics against thresholds"""
        alerts = []
        
        # Check OBS metrics
        alerts.extend(self._check_obs_metrics(obs_metrics))
        
        # Check system metrics
        alerts.extend(self._check_system_metrics(system_metrics))
        
        return alerts
    
    def _check_obs_metrics(self, metrics: Dict[str, Any]) -> List[Alert]:
        """Check OBS-specific metrics"""
        alerts = []
        
        # Dropped frames
        if "dropped_frames_percent" in metrics:
            value = metrics["dropped_frames_percent"]
            if value > self._thresholds.dropped_frames_percent:
                alerts.append(Alert(
                    severity=self._get_severity(value, self._thresholds.dropped_frames_percent),
                    alert_type="dropped_frames",
                    message=f"Dropped frames: {value:.1f}% (threshold: {self._thresholds.dropped_frames_percent}%)",
                    value=value,
                    threshold=self._thresholds.dropped_frames_percent,
                    source="obs"
                ))
        
        # FPS
        if "fps" in metrics:
            value = metrics["fps"]
            if value > 0 and value < self._thresholds.fps_drop_threshold:
                alerts.append(Alert(
                    severity=Severity.WARNING,
                    alert_type="low_fps",
                    message=f"Low FPS: {value:.1f} (threshold: {self._thresholds.fps_drop_threshold})",
                    value=value,
                    threshold=self._thresholds.fps_drop_threshold,
                    source="obs"
                ))
        
        return alerts
    
    def _check_system_metrics(self, metrics: Dict[str, Any]) -> List[Alert]:
        """Check system-wide metrics"""
        alerts = []
        
        # CPU
        if "cpu_percent" in metrics:
            value = metrics["cpu_percent"]
            if value > self._thresholds.cpu_percent:
                alerts.append(Alert(
                    severity=self._get_severity(value, self._thresholds.cpu_percent),
                    alert_type="high_cpu",
                    message=f"High CPU: {value:.1f}% (threshold: {self._thresholds.cpu_percent}%)",
                    value=value,
                    threshold=self._thresholds.cpu_percent,
                    source="system"
                ))
        
        # Memory
        if "memory_percent" in metrics:
            value = metrics["memory_percent"]
            if value > self._thresholds.memory_percent:
                alerts.append(Alert(
                    severity=self._get_severity(value, self._thresholds.memory_percent),
                    alert_type="high_memory",
                    message=f"High memory: {value:.1f}% (threshold: {self._thresholds.memory_percent}%)",
                    value=value,
                    threshold=self._thresholds.memory_percent,
                    source="system"
                ))
        
        return alerts
    
    def _get_severity(self, value: float, threshold: float) -> Severity:
        """Determine severity based on how much threshold is exceeded"""
        ratio = value / threshold
        
        if ratio >= 1.5:
            return Severity.CRITICAL
        elif ratio >= 1.2:
            return Severity.WARNING
        else:
            return Severity.INFO
    
    def should_alert(self, alert_type: str) -> bool:
        """Check if we should alert (respect suppression period)"""
        config = get_config()
        suppression = config.alerts.suppression_seconds
        
        now = datetime.now()
        
        if alert_type in self._last_alert_time:
            last_time = self._last_alert_time[alert_type]
            if (now - last_time).total_seconds() < suppression:
                return False
        
        self._last_alert_time[alert_type] = now
        return True
    
    def reset_suppression(self) -> None:
        """Reset all suppression timers"""
        self._last_alert_time.clear()
"""Tests for collectors"""
import pytest
from unittest.mock import Mock, AsyncMock, patch

from stream_monitor.collectors.base import BaseCollector, CollectorMetrics
from stream_monitor.collectors.obs import OBSCollector
from stream_monitor.collectors.system import SystemCollector


class TestCollectorMetrics:
    def test_to_dict(self):
        from datetime import datetime
        metrics = CollectorMetrics(
            timestamp=datetime.now(),
            source="test",
            data={"value": 42}
        )
        result = metrics.to_dict()
        assert result["source"] == "test"
        assert result["data"]["value"] == 42


class TestOBSCollector:
    @pytest.mark.asyncio
    async def test_init(self):
        collector = OBSCollector(poll_interval=2)
        assert collector.name == "obs"
        assert collector.poll_interval == 2
    
    @pytest.mark.asyncio
    async def test_health_check_no_client(self):
        collector = OBSCollector()
        result = await collector.health_check()
        assert result is False


class TestSystemCollector:
    @pytest.mark.asyncio
    async def test_init(self):
        collector = SystemCollector(poll_interval=5)
        assert collector.name == "system"
        assert collector.poll_interval == 5
    
    @pytest.mark.asyncio
    async def test_collect(self):
        collector = SystemCollector()
        await collector.connect()
        metrics = await collector.collect()
        
        assert metrics.source == "system"
        assert "cpu_percent" in metrics.data
        assert "memory_percent" in metrics.data
    
    @pytest.mark.asyncio
    async def test_health_check(self):
        collector = SystemCollector()
        await collector.connect()
        result = await collector.health_check()
        assert result is True
"""Tests for storage"""
import pytest
import tempfile
import os
from datetime import datetime, timedelta
from pathlib import Path

from stream_monitor.storage.database import Database, HealthMetrics, StreamAlert
from stream_monitor.storage.aggregator import MetricsAggregator


class TestDatabase:
    @pytest.fixture
    def temp_db(self):
        with tempfile.TemporaryDirectory() as tmpdir:
            db_path = os.path.join(tmpdir, "test.db")
            yield db_path
    
    def test_init(self, temp_db):
        db = Database(temp_db)
        assert db._db_path == temp_db
    
    def test_save_metrics(self, temp_db):
        db = Database(temp_db)
        db.connect()
        
        db.save_metrics("test_source", {"metric1": 100, "metric2": 50.5})
        
        metrics = db.get_recent_metrics("test_source", minutes=5)
        assert len(metrics) > 0
        
        db.close()
    
    def test_create_alert(self, temp_db):
        db = Database(temp_db)
        db.connect()
        
        alert = db.create_alert(
            severity="WARNING",
            alert_type="test_alert",
            message="Test message",
            value=75.0,
            threshold=80.0
        )
        
        assert alert.severity == "WARNING"
        assert alert.alert_type == "test_alert"
        
        db.close()
    
    def test_cleanup_old_data(self, temp_db):
        db = Database(temp_db)
        db.connect()
        
        # Should not raise error
        deleted = db.cleanup_old_data(retention_hours=24)
        assert deleted >= 0
        
        db.close()


class TestMetricsAggregator:
    def test_init(self):
        aggregator = MetricsAggregator()
        assert aggregator is not None
    
    def test_add_metric(self):
        aggregator = MetricsAggregator()
        aggregator.add_metric("test", {"value": 42})
        assert len(aggregator._raw_metrics) > 0
    
    def test_calculate_moving_average(self):
        aggregator = MetricsAggregator()
        
        # Add some metrics
        for i in range(10):
            aggregator.add_metric("test", {"value": float(i)})
        
        avg = aggregator.calculate_moving_average("test", "value", window_minutes=1)
        assert avg >= 0
# Stream Health Monitor Configuration

# OBS WebSocket Configuration
obs:
  host: "localhost"
  port: 4455
  password: ""
  reconnect_interval: 5
  max_reconnect_attempts: 10

# System Metrics Configuration
system:
  poll_interval: 5  # seconds
  obs_process_monitoring: true

# Storage Configuration
storage:
  database_path: "data/stream_health.db"
  retention_hours: 24
  aggregation_intervals:
    - 5    # 5-second buckets
    - 60   # 1-minute buckets
    - 300  # 5-minute buckets

# Alerting Configuration
alerts:
  enabled: true
  suppression_seconds: 300  # 5 minutes
  
  thresholds:
    dropped_frames_percent: 5
    cpu_percent: 85
    memory_percent: 90
    bitrate_variance_percent: 15
    network_latency_ms: 200
    fps_drop_threshold: 5

  severity_levels:
    - INFO
    - WARNING
    - CRITICAL

# Notification Channels
notifications:
  console:
    enabled: true
    level: "INFO"
  
  file:
    enabled: true
    path: "logs/stream_health.log"
    level: "DEBUG"
  
  webhook:
    enabled: false
    url: ""
    level: "WARNING"

# Platform Integration (Optional)
platform:
  douyin:
    enabled: false
    poll_interval: 30
  
  bili:
    enabled: false
    poll_interval: 30

# StreamerAI Integration
streamer_ai:
  enabled: false
  database_path: "../StreamerAI/data/test.db"
  control_table: "monitor_commands"
[tool.poetry]
name = "stream-health-monitor"
version = "0.1.0"
description = "Standalone Stream Health Monitoring Tool"
authors = ["StreamerAI Team"]
readme = "README.md"
packages = [{include = "stream_monitor"}]

[tool.poetry.dependencies]
python = "^3.10"
peewee = "^3.17.0"
psutil = "^5.9.0"
obsws-python = "^1.7.0"
rich = "^13.7.0"
pyyaml = "^6.0.0"
aiohttp = "^3.9.0"

[tool.poetry.dependencies.streamer-ai]
version = "^0.1.0"
optional = true

[tool.poetry.extras]
platform = ["playwright", "blivedm"]
integration = ["streamer-ai"]

[tool.poetry.scripts]
stream-monitor = "stream_monitor.main:main"
stream-monitor-dashboard = "stream_monitor.ui.cli_dashboard:main"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
pytest-cov = "^4.1.0"
# Stream Health Monitor

Standalone Python-based stream health monitoring tool that tracks OBS stream quality, system resources, network metrics, and platform analytics. Can optionally integrate with StreamerAI via shared SQLite database for coordinated alerts and control.

## Features

- **OBS WebSocket Integration**: Real-time FPS, bitrate, dropped frames, CPU/memory monitoring
- **System Metrics**: CPU, memory, disk I/O, network interfaces via psutil
- **Threshold Alerts**: Configurable thresholds with INFO/WARNING/CRITICAL severity
- **Multiple Notification Channels**: Console, file logs, Discord/Slack webhooks
- **Trend Analysis**: Moving averages, anomaly detection, issue prediction
- **Optional Platform Integration**: Douyin and Bilibili viewer metrics
- **StreamerAI Integration**: Coordinate AI behavior during stream issues

## Installation

```bash
# Clone the repository
cd StreamHealthMonitor

# Install dependencies with Poetry
poetry install

# Or with pip
pip install -e .
```

## Quick Start

### With Poetry
```bash
poetry install
poetry run stream-monitor
poetry run stream-monitor-dashboard
```

### With pip
```bash
pip install -e .
python -m stream_monitor.main
python -m stream_monitor.ui.cli_dashboard
```

### Using a custom config file
```bash
python -m stream_monitor.main config.yaml
```

## Configuration

Copy `config.yaml.example` to `config.yaml` and modify as needed:

```yaml
obs:
  host: "localhost"
  port: 4455
  password: ""

alerts:
  thresholds:
    dropped_frames_percent: 5
    cpu_percent: 85
    memory_percent: 90

notifications:
  console:
    enabled: true
  webhook:
    enabled: true
    url: "https://discord.com/api/webhooks/..."
```

## Architecture

```
stream_monitor/
├── collectors/       # Data collectors (OBS, system, platform)
├── storage/         # Database and aggregation
├── alerting/        # Threshold checking and notifications
├── integration/     # StreamerAI bridge
└── ui/              # CLI and web dashboards
```

## Modes

1. **Standalone**: Monitor OBS + system metrics only
2. **Platform**: Add `--extras platform` for Douyin/Bili
3. **StreamerAI**: Add `--extras integration` for AI coordination

## License

MIT
"""Setup configuration for pip installation"""
from setuptools import setup, find_packages

setup(
    name="stream-health-monitor",
    version="0.1.0",
    description="Standalone Stream Health Monitoring Tool",
    author="StreamerAI Team",
    packages=find_packages(),
    python_requires=">=3.10",
    install_requires=[
        "peewee>=3.17.0",
        "psutil>=5.9.0",
        "obsws-python>=1.7.0",
        "rich>=13.7.0",
        "pyyaml>=6.0.0",
        "aiohttp>=3.9.0",
    ],
    extras_require={
        "platform": ["playwright", "blivedm"],
        "dev": ["pytest>=7.4.0", "pytest-asyncio>=0.21.0", "pytest-cov>=4.1.0"],
    },
    entry_points={
        "console_scripts": [
            "stream-monitor=stream_monitor.main:main",
            "stream-monitor-dashboard=stream_monitor.ui.cli_dashboard:main",
        ],
    },
    classifiers=[
        "Development Status :: 3 - Alpha",
        "Intended Audience :: Developers",
        "Programming Language :: Python :: 3",
        "Programming Language :: Python :: 3.10",
        "Programming Language :: Python :: 3.11",
        "Programming Language :: Python :: 3.12",
    ],
)
