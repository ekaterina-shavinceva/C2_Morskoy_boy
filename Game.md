# C2_Morskoy_boy
from random import randint

# класс для управления точек на поле
class Dot:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
    def __repr__(self):
        return f'({self.x}, {self.y})'

#создаем исклюения
class BoardException(Exception):
    pass

class BoardOutException(BoardException):
    def __str__(self):
        return 'Вы пытаетесь выстрелить за доску!'

class BoardUsedException(BoardException):
    def __str__(self):
        return 'Вы уже стреляли в эту клетку'

class BoardWrongShipException(BoardException):
    pass


# создаем класс корабль

class Ship:
    def __init__(self, bow, l, o):
        self.bow = bow
        self.l = l
        self.o = o
        self.lives = l

    @property
    def dots(self):
        ship_dots = []
        for i in range(self.l):
            cur_x = self.bow.x
            cur_y = self.bow.y

            if self.o == 0:
                cur_x += i
            elif self.o == 1:
                cur_y += i

            ship_dots.append(Dot(cur_x, cur_y))
        return ship_dots

    def shooten(self, shot):
        return shot in self.dots




# создаем класс игровое поле, size - размер игрового поля, hid - нужно ли скрывать корабли на доске
class Board:
    def __init__(self, hid = False, size = 6):
        self.size = size
        self.hid = hid

        self.count = 0

        self.field = [['0'] * size for i in range(size)]

        # заводим переменные на занятые ячейки и ячеки с кораблем
        # busy - уже был выстрел в эту ячейку,
        # ships - ячейка с кораблем
        self.busy = []
        self.ships = []

    def add_ship(self, ship):
        for d in ship.dots:
            if self.out(d) or d in self.busy:
                raise BoardWrongShipException()
        for d in ship.dots:
            self.field[d.x][d.y] = '■'
            self.busy.append(d)

        self.ships.append(ship)
        self.contur(ship)

    # добавление на доску контура корабля
    # near - ближняя к кораблю ячейка
    def contur(self, ship, verb = False):
        near = [(-1, -1), (-1, 0), (-1, 1),
                (0, -1), (0, 0), (0, 1),
                (1, -1), (1, 0), (1, 1)]

        for d in ship.dots:
            for d_x, d_y in near:
                cur = Dot(d.x + d_x, d.y + d_y)
                if not (self.out(cur)) and cur not in self.busy:
                    if verb:
                        self.field[cur.x][cur.y] = 'T'
                    self.busy.append(cur)



    def __str__(self):
        res = ''
        res += '  | 1 | 2 | 3 | 4 | 5 | 6 |'
        for i, line in enumerate(self.field):
            res += f'\n{i+1} | {" | ".join(line)} | '

        if self.hid:
            res = res.replace('■', '0')
        return res

    def out(self, d):
        return  not((0 <= d.x < self.size) and (0 <= d.y < self.size))


# методы стрельбы по доске
    def shot(self, d):
        if self.out(d):  # есил выстрел мимо доски, вызываем исключение BoardOutException
            raise BoardOutException()
        if d in self.busy: # если повторный выстрел в ячейку, вызываем исключение BoardUsedException
            raise BoardUsedException()

        self.busy.append(d)

        for ship in self.ships:
            if d in ship.dots: # если выстрел попал в корабль, минус 1 жизнь и отмечаем попадпние Х
                ship.lives -= 1
                self.field[d.x][d.y] = 'X'
                if ship.lives == 0: # если все части корабля уничтожены, счетчик кораблей плюс, и выводим информацию об иничтожении
                    self.count += 1
                    self.contur(ship, verb = True)
                    print('Корабль уничтожен!')
                    return True # повтор хода при уничтожении корабля
                else:              
                    print('Корабль ранен!')
                    return True # повтор хода при ранении

            self.field[d.x][d.y] = 'T'
            print('Мимо!')
            return False
    def begin(self):
        self.busy = []

# создаем класс Игрока
class Player:
    def __init__(self, board, enemy):
        self.board = board
        self.enemy = enemy

    def ask(self):
        raise NotImplementedError()

    def move(self):
        while True:
            try:
                target = self.ask()
                repeat = self.enemy.shot(target)
                return repeat
            except BoardException as e:
                print(e)


# создаем класс Игрок-ПК и Игрок-Пользователь
class AI(Player): # класс Игрока-ПК, дочерний от класса Player
    def ask(self):
        d = Dot(randint(0,5), randint(0,5))
        print(f'Ход компьютера: {d.x + 1} {d.y + 1}')
        return d

class User(Player):# класс Игрока-Пользователя, дочерний от класса Player
    def ask(self):
        while True:
            cords = input('Ваш ход! (введите две координаты от 1 до 6 через пробел): ').split()
            if len(cords) != 2:
                print('Введите две координаты через пробел!')
                continue

            x, y = cords

            if not(x.isdigit()) or not(y.isdigit()): # проверям тип введенных символов, должны быть числа
                print('Некорректный ввод, введите два числа координат через пробел!')
                continue

            x, y = int(x), int(y)

            if x < 1 or x > 6 or y < 1 or y > 6:
                print('Координаты вне диапазона! Введите числа от 1 до 6!')
                continue

            return Dot(x-1, y-1)

# создаем класс игры и генерации доски
class Game:
    def __init__(self, size = 6):
        self.size = size
        pl = self.random_board()
        co = self.random_board()
        co.hid = True

        self.ai = AI(co, pl)
        self.us = User(pl, co)

    def random_board(self):
        board = None
        while board is None:
            board = self.try_boars()
        return board

    def try_boars(self):
        lens = [3, 2, 2, 1, 1, 1, 1] # создаем список с количеством кораблей
        board = Board(size = self.size)
        attempts = 0
        for l in lens:
            while True:
                attempts += 1
                if attempts > 2000:
                    return None
                ship = Ship(Dot(randint(0, self.size), randint(0, self.size)) , l, randint(0,1))
                try:
                    board.add_ship(ship)
                    break
                except BoardWrongShipException:
                    pass
        board.begin()
        return board



    # создаем функцию приветсвия и краткой инструкции
    def manual(self):
        print('~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~')
        print('Добро пожаловать в игру "Моской бой"!')
        print('~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~')
        print()
        print('Формат ввода: x y ')
        print('в диапозоне от 0 до 2 через пробел')
        print('x - номер строки, y - номер столбца')
        print('~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~')

    def loop(self):
        num = 0
        while True:
            print('~' * 20)
            print('Доска пользователя')
            print(self.us.board)
            print('~' * 20)
            print('Доска компьютера')
            print(self.ai.board)
            if num % 2 == 0:
                print('~' * 20)
                print('Ходит пользователь')
                repeat = self.us.move()
            else:
                print('~' * 20)
                print('Ходит компьютер')
                repeat = self.ai.move()
            if repeat:
                num -= 1

            if self.ai.board.count == 7:
                print('~' * 20)
                print('Пользователь выиграл!!!')
                break

            if self.us.board.count == 7:
                print('~' * 20)
                print('Компьютер выиграл!!!')
                break
            num += 1

    def start(self):
        self.manual()
        self.loop()

g = Game()
g.start()














