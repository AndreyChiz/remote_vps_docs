# Backend сервис chiz.work.gd

[Документация]()

### Systemd

/etc/systemd/system/chiz_backend_fastapi.service

```conf
[Unit]
Description=FastAPI backend service for chiz.work.gb
After=network.target

[Service]
User=achi
Group=achi
WorkingDirectory=/home/www/src/chiz.work.gb-backend
ExecStart=/home/www/src/chiz.work.gb-backend/.venv/bin/gunicorn app.main:app \
          --workers 8 \
          --worker-class uvicorn.workers.UvicornWorker \
          --bind 0.0.0.0:8001 \
          --access-logfile /home/www/log/chiz.work.gd/gunicorn_back/chiz.work.gb_access.log \
          --error-logfile /home/www/log/chiz.work.gd/gunicorn_back/chiz.work.gb_error.log \
          --log-level info \
          --timeout 60
Restart=always
RestartSec=3
Environment="PATH=/home/www/src/chiz.work.gb-backend/.venv/bin:/usr/bin"

[Install]
WantedBy=multi-user.target



```
 
