# Algorithmic Trading Starter Pack

- `algorithmic_trading_guide.pdf`: 요청사항을 반영한 전체 가이드 PDF
- `guide.md`: PDF 원문(Markdown)
- `trading_system.py`: 페이퍼트레이딩 기본 + 실거래 전환 가능한 엔진 템플릿
- `config.example.yaml`: 실행 설정 예시
- `requirements.txt`: 의존성

## Quick Start

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp config.example.yaml config.yaml
python trading_system.py
```
