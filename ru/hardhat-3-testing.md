---
title: Тестируем смарт-контракты на Hardhat 3: node:test, viem и ни одного mocha
slug: hardhat-3-testing
lang: ru
date: '2026-09-15'
description: Hardhat в третьей версии решили не мелочиться, взяли и сменили весь тестовый стек. Вместо привычной связки mocha + chai + ethers теперь такой родной node:test и viem.…
canonical: https://habr.com/ru/articles/1082316/
---

Hardhat в третьей версии решили не мелочиться, взяли и сменили весь тестовый стек. Вместо привычной связки mocha + chai + ethers теперь такой родной `node:test` и `viem`. Апгрейд-гайды в основном рассказывают, как мигрировать конфиг, а вот как теперь **писать тесты** - материала заметно меньше. Я не так давно, буквально несколько дней назад, писал свой учебный проект на Hardhat 3 и собрал рецепт целиком. От init до ассертов на балансы. Всё из этой статьи ты сможешь повторить за вечер

Тестировать будем мой учебный проект контракт-копилку. Его суть максимально простая. Положить деньги в копилку могут все, забирать может только владелец, и только тогда, когда накоплена цель. Полный код [в репозитории](https://github.com/blockwright/kopilka), но для статьи достаточно его интерфейса
```
function deposit() external payable;   // revert EmptyDeposit()
function withdraw() external;          // revert NotOwner() / GoalNotReached(current, goal)
function progress() external view returns (uint256 current, uint256 target);

event Deposited(address indexed from, uint256 amount, uint256 total);
event Withdrawn(address indexed to, uint256 amount);
```
### Что изменилось в Hardhat 3

| Было (HH2) | Стало (HH3) |
| --- | --- |
| mocha | `node:test` - раннер из коробки Node |
| chai + chai-matchers | `viem.assertions` |
| ethers | viem |
| `hre.ethers.getSigners()` | `viem.getWalletClients()` |
| хардкодная сеть | `network.create()` - своя сеть на describe |

Лично мой фаворит среди приятных плюшек - это обычный TypeScript без глобальной магии. Из того, к чему придется немного привыкать - часть API стала асинхронной там, где раньше была синхронной.

Требования: Node 22.13+ (жёсткое, `node:test` нужной версии), дальше:
```
npx hardhat --init   # шаблон: node-test-runner-viem
```
Если выбрал шаблон, как из примера выше, тогда у тебя есть `contracts/`, `test/`, `ignition/` и конфиг на TypeScript. Пуляем контракт в `contracts/` и полетели тестить.

### Каркас теста
```
import assert from "node:assert/strict";
import { describe, it } from "node:test";
import { parseEther, getAddress } from "viem";
import { network } from "hardhat";

describe("Kopilka", async function () {
const { viem } = await network.create();
const [owner, stranger] = await viem.getWalletClients();

const GOAL = parseEther("1");
const deploy = () => viem.deployContract("Kopilka", [GOAL]);

// тесты ниже
});
```
Две вещи, которые, на мой взгляд, стоит посмотреть дважды. Первая это `describe`, и он асинхронный, а вместе с ним `network.create()`, который живёт внутри него. Каждый describe получает собственную EVM в памяти. "Зачем это нужно?" - спросишь ты. Отвечаю: мгновенные блоки, 20 предзаряженных аккаунтов, ноль сетевых вызовов. Вторая вещь это `deploy` и он стал фабрикой. И снова тот же вопрос от тебя и снова ответ от меня: Свежий контракт на каждый тест дешевле, чем отлов межтестового состояния между тестами и попытка понять, "А как так выходит, что тест вроде правильный, а падает с кривыми цифрами?"

Первый тест — тривиальный, но он проверяет весь конвейер от компиляции до чтения:
```
it("деплой: владелец — деплоер, цель записана", async function () {
const kopilka = await deploy();

assert.equal(await kopilka.read.owner(), getAddress(owner.account.address));
const [current, target] = await kopilka.read.progress();
assert.equal(current, 0n);
assert.equal(target, GOAL);
});
```
Запоминаем, где зарыта первая грабля: `read.owner()` возвращает адрес в EIP-55 checksum-регистре, а `owner.account.address` в нижнем. Из этого следует, что сравнивать надо через `getAddress()`, иначе тест падает на «одинаковых» адресах.

### События: emitWithArgs
```
it("deposit эмитит Deposited", async function () {
const kopilka = await deploy();

await viem.assertions.emitWithArgs(
kopilka.write.deposit({ value: parseEther("0.4"), account: stranger.account }),
kopilka,
"Deposited",
[getAddress(stranger.account.address), parseEther("0.4"), parseEther("0.4")],
);
});
```
Тут все довольно стандартно. Аргументы сверяются позиционно по ABI, не перепутай порядок. Обрати внимание на третий аргумент. Контракт пишет в событие `address(this).balance`, а он к моменту emit **уже включает** пришедший депозит. Небольшой вывод из опыта: Тест на событие - это прекрасный способ зафиксировать тонкости и сложности семантики контракта, о которых не расскажет ни один геттер. Поменяешь контракт и упадет тест из-за несоответствия нового ABI и аргументов теста.

### Отказы: revertWithCustomError

Тест «функция работает» пишут все. Давай говорить откровенно: на такой "сложный" тест ума много не надо. Но в жизни все наоборот. Деньги теряют там, где функция **не отказала**, хотя должна была:
```
it("withdraw чужаком откатывается", async function () {
const kopilka = await deploy();
await kopilka.write.deposit({ value: GOAL });

await viem.assertions.revertWithCustomError(
kopilka.write.withdraw({ account: stranger.account }),
kopilka,
"NotOwner",
);
});
```
Теперь под другим углом, тот же профиль. Версия с аргументами, которая проверяет не только причину отказа, но и цифры в ней:
```
await viem.assertions.revertWithCustomErrorWithArgs(
kopilka.write.withdraw(),
kopilka,
"GoalNotReached",
[parseEther("0.3"), GOAL],
);
```
И опять из опыта: это намного сильнее, чем может казаться: «отказал, но не по той причине» - это полноценный баг, который строковые require-сообщения ловили плохо, а типизированные custom errors ловят бесплатно и главное "красиво".

### Деньги: balancesHaveChanged

Мой любимый ассерт нового стека:
```
it("после цели владелец забирает всё", async function () {
const kopilka = await deploy();
await kopilka.write.deposit({ value: GOAL, account: stranger.account });

await viem.assertions.balancesHaveChanged(kopilka.write.withdraw(), [
{ address: owner.account.address, amount: GOAL },
]);
});
```
Здесь одна строка кода утверждает главное для любого контракта с деньгами. Проверяем, что после транзакции баланс владельца вырос ровно на GOAL. Что мне нравится здесь особенно, так это то, что комиссию за газ ассерт учитывает сам, не нужно ничего вычитать руками и следить за математикой лишний раз. Типичное место кривых тестов - плохо посчитанная вручную математика, при их написании. В HH2 такой тест собирался из getBalance-до и getBalance-после и ручной арифметики с receipt.gasUsed. Теперь это просто хорошо забытое старое.

### Шпаргалка по viem.assertions

| Ассерт | Проверяет |
| --- | --- |
| `revert(fn)` | откат по любой причине |
| `revertWith(fn, "reason")` | откат со строкой из require |
| `revertWithCustomError(fn, c, "Err")` | откат с конкретной custom error |
| `revertWithCustomErrorWithArgs(fn, c, "Err", [...])` | …и с конкретными аргументами |
| `emit(tx, c, "Event")` | событие было |
| `emitWithArgs(tx, c, "Event", [...])` | событие с аргументами (позиционно) |
| `balancesHaveChanged(tx, [{address, amount}])` | дельты ETH-балансов с учетом газа |

### Опыт, собранный по дороге

1.

**Checksum-адреса** - сравнивай через `getAddress()`

2.

`**describe**`** обязан быть async**, а `network.create()` внутри него

3.

**Всё в bigint.** `assert.equal(current, 0)` упадёт: `0 !== 0n`. Литералы с суффиксом `n`, суммы только через `parseEther`

4.

**Node ниже 22.13** - самая частая причина «ничего не работает» сразу после init. Ошибки поймаешь не про версию ноды, а про синтаксис, так что может быть не очевидно

Запуск - `npx hardhat test`. Семь тестов моей копилки пробегают за треть секунды, потому что сеть живёт в памяти процесса. CI-конфиг сводится к `npm ci && npx hardhat test` - никаких сервисов и RPC-ключей. Не это ли счастье?

Полный тест-файл и контракт — [в репо](https://github.com/blockwright/kopilka), тег `post-12`. Если соберёте по этому рецепту свой набор, буду рад услышать, какой ассерт окажется самым ходовым у вас. Ну и не забывайте подписываться на мой телеграм, ссылка в профиле, там каждый день посты, словари и впереди только больше и лучше.

-

**_Это Райт. Ни бага, ни костыля тебе, друг!_**
