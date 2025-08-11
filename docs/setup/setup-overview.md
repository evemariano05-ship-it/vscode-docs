---
ContentId: <!doctype html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Jogo: Clínica de Fisioterapia</title>
  <style>
    :root{font-family:Inter, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;}
    body{background:#f3f6fb;color:#0b2545;display:flex;align-items:center;justify-content:center;height:100vh;margin:0;padding:20px}
    .card{background:#fff;border-radius:12px;box-shadow:0 8px 24px rgba(11,37,69,0.08);width:920px;max-width:100%;padding:22px}
    h1{margin:0 0 8px;font-size:20px}
    .top{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}
    .stat{background:#eef5ff;padding:10px;border-radius:8px;font-weight:600}
    .client{background:#fff7e6;padding:14px;border-radius:10px;margin-bottom:12px}
    .options{display:grid;grid-template-columns:repeat(2,1fr);gap:12px}
    button.option{padding:14px;border-radius:10px;border:2px solid #dbe9ff;background:white;cursor:pointer;font-weight:600}
    button.option:hover{transform:translateY(-2px)}
    .footer{display:flex;justify-content:space-between;align-items:center;margin-top:14px}
    .small{font-size:13px;color:#5b6b85}
    .next{background:#0b6bff;color:white;border:none;padding:10px 14px;border-radius:8px;cursor:pointer;font-weight:700}
    .disabled{opacity:0.5;pointer-events:none}
    .result{margin-top:12px;padding:10px;border-radius:8px;background:#f0fff5;color:#064e2a}
    .wrong{background:#fff6f6;color:#7a1b1b}
    .center{text-align:center}
    .controls{display:flex;gap:8px}
    .small-btn{padding:8px 10px;border-radius:8px;border:1px solid #d0d7e8;background:#fff;cursor:pointer}
    footer.note{margin-top:12px;font-size:13px;color:#6b7b93}
  </style>
</head>
<body>
  <div class="card">
    <div class="top">
      <h1>Clínica de Fisioterapia — Simulador</h1>
      <div style="display:flex;gap:8px;align-items:center">
        <div class="stat">Faturamento: <span id="revenue">R$ 0,00</span></div>
        <div class="stat">Atendimentos: <span id="round">0</span></div>
      </div>
    </div>

    <div class="client">
      <div class="small">Próximo paciente:</div>
      <div id="diagnosis" style="font-size:18px;font-weight:700;margin-top:6px">Carregando...</div>
      <div class="small" style="margin-top:6px">Escolha o alongamento mais indicado — cada opção vale um valor em reais (padrão: acerto = R$10).</div>
    </div>

    <div class="options" id="options"></div>

    <div id="feedback"></div>

    <div class="footer">
      <div class="controls">
        <button class="small-btn" id="btn-next">Próximo paciente</button>
        <button class="small-btn" id="btn-restart">Reiniciar</button>
      </div>
      <div class="small">Rodadas totais: <span id="totalRounds">10</span></div>
    </div>

    <footer class="note">
      Dicas: você pode editar a lista de diagnósticos no código para incluir mais casos. Quer que eu torne este jogo mais complexo (tempo limite, nível, moedas)? Me pede que eu faço as alterações.
    </footer>
  </div>

<script>
// --- CONFIGURAÇÕES ---
const TOTAL_ROUNDS = 10; // quantos pacientes por jogo
// valores possíveis (de R$10 a R$0). Padrão: acerto = 10, demais valores decrescentes.
const VALUES = [10, 7, 3, 0];

// Base de dados de pacientes: cada caso tem diagnóstico e alongamento correto.
// Você pode editar/adicionar casos aqui.
const CASES = [
  {diagnosis: 'Contratura do trapézio superior (dor cervical e ombro)', correct: 'Alongamento do trapézio superior (inclinação lateral da cabeça)'} ,
  {diagnosis: 'Síndrome do túnel do carpo', correct: 'Alongamento dos flexores do punho com extensão do punho e dedos'} ,
  {diagnosis: 'Tendinopatia do bíceps braquial', correct: 'Alongamento do bíceps em extensão de cotovelo e ombro'} ,
  {diagnosis: 'Cefaleia tensional por tensão cervical', correct: 'Alongamento cervical global (flexão cervical e alongamento do esternocleidomastoideo)'} ,
  {diagnosis: 'Dor lombar mecânica (musculatura paravertebral tensa)', correct: 'Alongamento dos paravertebrais lombares e flexão de tronco'} ,
  {diagnosis: 'Síndrome do desfiladeiro torácico', correct: 'Alongamento do peitoral maior e menor (abdução e rotação externa do ombro)'} ,
  {diagnosis: 'Iliotibial friccional (joelho de corredor)', correct: 'Alongamento do trato iliotibial e tensor da fáscia lata'} ,
  {diagnosis: 'Pubalgia/ dor adutora', correct: 'Alongamento dos adutores com abertura de pernas (abdução passiva)'} ,
  {diagnosis: 'Tendinopatia do supraespinhal (ombro)', correct: 'Alongamento do manguito rotador: rotação interna/externa e alongamento do supraespinhal'} ,
  {diagnosis: 'Contratura do músculo piriforme (dor referida ao glúteo/ciático)', correct: 'Alongamento do piriforme sentado ou deitado (flexão e rotação de quadril)'}
];

// Lista de descrições de alongamentos (pool para gerar alternativas)
const STRETCH_POOL = Array.from(new Set(CASES.map(c=>c.correct).concat([
  'Alongamento dos isquiotibiais sentado',
  'Alongamento do quadríceps em pé',
  'Alongamento do peitoral em porta',
  'Alongamento do piriforme em pé',
  'Alongamento dos flexores do quadril com retroversão pélvica',
  'Mobilização ativa do ombro em rotação',
  'Alongamento do tríceps com braço atrás da cabeça',
  'Alongamento do latíssimo do dorso (alongamento lateral)',
  'Alongamento dos flexores do punho com palma para baixo',
  'Alongamento do glúteo em decúbito (abraçar joelho)'
]));

// --- ESTADO ---
let remainingCases = [];
let currentCase = null;
let round = 0;
let revenue = 0;

// --- UTILIDADES ---
function formatMoney(v){ return 'R$ ' + v.toFixed(2).replace('.',','); }
function shuffle(a){ for(let i=a.length-1;i>0;i--){ const j=Math.floor(Math.random()*(i+1)); [a[i],a[j]]=[a[j],a[i]] } return a }

// --- INICIALIZAÇÃO ---
function init(){
  document.getElementById('totalRounds').textContent = TOTAL_ROUNDS;
  resetGame();
  document.getElementById('btn-next').addEventListener('click', nextPatient);
  document.getElementById('btn-restart').addEventListener('click', resetGame);
}

function resetGame(){
  remainingCases = shuffle(CASES.slice());
  round = 0;
  revenue = 0;
  updateUI();
  nextPatient();
}

function updateUI(){
  document.getElementById('revenue').textContent = formatMoney(revenue);
  document.getElementById('round').textContent = round;
}

function nextPatient(){
  clearFeedback();
  if(round >= TOTAL_ROUNDS){
    showFinal();
    return;
  }
  round++;
  if(remainingCases.length === 0) remainingCases = shuffle(CASES.slice());
  currentCase = remainingCases.pop();
  showCase(currentCase);
  updateUI();
}

function showCase(c){
  document.getElementById('diagnosis').textContent = c.diagnosis;
  // montar 3 alternativas erradas aleatórias
  let wrongs = STRETCH_POOL.filter(s => s !== c.correct);
  shuffle(wrongs);
  let options = [c.correct, wrongs[0], wrongs[1] || wrongs[0], wrongs[2] || wrongs[0]].slice(0,4);
  options = shuffle(options);

  // atribuição de valores: por padrão acerto = 10, outros = 7,3,0
  // Os valores seguem abertura fixa para clareza. Se quiser, posso fazer um modo embaralhado.
  const values = VALUES.slice(); // [10,7,3,0]

  // Ao mostrar, vamos atribuir os valores de forma aleatória para as opções? NÃO — manteremos acerto=10 para ensino.
  // Se quiser modo com valores embaralhados, peça que eu implemente.

  const container = document.getElementById('options');
  container.innerHTML = '';
  options.forEach(optText => {
    const btn = document.createElement('button');
    btn.className = 'option';
    // se for a opção correta, garanto valor 10, caso contrário pego próximos valores
    const isCorrect = optText === c.correct;
    let value = 0;
    if(isCorrect) value = 10;
    else {
      // distribuir os valores restantes (7,3,0) de forma determinística segundo índice para evitar empate visual
      const idx = options.indexOf(optText);
      // idx pode ser 0..3 — se idx==index of correct, precisa ajustar
      const otherValues = [7,3,0];
      // achar posição entre os 3 errados
      const wrongIndex = options.filter(o=>o!==c.correct).indexOf(optText);
      value = otherValues[wrongIndex] || 0;
    }

    btn.innerHTML = `<div style=\"text-align:left\">${optText}</div><div style=\"margin-top:8px;font-size:14px;font-weight:800;\">Valor: R$ ${value},00</div>`;
    btn.addEventListener('click', ()=> chooseOption(optText, value, isCorrect, btn));
    container.appendChild(btn);
  });
}

function chooseOption(text, value, correct, btn){
  // bloquear todas opções
  document.querySelectorAll('.option').forEach(b=>b.classList.add('disabled'));
  // adicionar faturamento
  revenue += value;
  updateUI();

  const feedback = document.getElementById('feedback');
  if(correct){
    feedback.innerHTML = `<div class=\"result\">Acertou! Você faturou ${formatMoney(value)} neste atendimento.</div>`;
    btn.style.borderColor = '#0bb26a';
  } else {
    feedback.innerHTML = `<div class=\"wrong\">Resposta incorreta. Você recebeu ${formatMoney(value)} (valor do item escolhido).</div><div style=\"margin-top:8px;color:#23304d\">Resposta correta: <strong>${currentCase.correct}</strong></div>`;
    btn.style.borderColor = '#d94a4a';
    // destacar a correta
    document.querySelectorAll('.option').forEach(b => {
      if(b.innerText.includes(currentCase.correct)) b.style.borderColor = '#0bb26a';
    });
  }
}

function clearFeedback(){ document.getElementById('feedback').innerHTML = ''; }

function showFinal(){
  const container = document.getElementById('options');
  container.innerHTML = '<div class="center" style="grid-column:1/-1"><h2>Jogo finalizado</h2><p style="font-weight:700">Faturamento total: '+formatMoney(revenue)+'</p><p class="small">Clique em Reiniciar para jogar novamente.</p></div>';
  document.getElementById('diagnosis').textContent = 'Nenhum paciente por enquanto';
}

// Inicializa a interface
init();
</script>
</body>
</html>
FC5262F3-D91D-4665-A5D2-BCBCCF66E53A
DateApproved: 08/07/2025
MetaDescription: Get Visual Studio Code up and running.
MetaSocialImage: images/quicksetup/quick-setup-social.png
---
# Setting up Visual Studio Code

VS Code is a free code editor, which runs on the macOS, Linux, and Windows operating systems. Getting up and running with Visual Studio Code is quick and easy. It is a small download so you can install in a matter of minutes and give VS Code a try.

VS Code is lightweight and should run on most available hardware and platform versions. You can review the [System Requirements](/docs/supporting/requirements.md) to check if your computer configuration is supported.

## Set up VS Code for your platform

1. Download and install Visual Studio Code for your platform

    * [macOS](/docs/setup/mac.md)
    * [Linux](/docs/setup/linux.md)
    * [Windows](/docs/setup/windows.md)

    > [!NOTE]
    > VS Code ships monthly releases and supports [auto-update](#update-cadence) when a new release is available.

1. [Install additional components](/docs/setup/additional-components.md)

    Install Git, Node.js, TypeScript, language runtimes, and more.

1. [Install VS Code extensions from the Visual Studio Marketplace](https://marketplace.visualstudio.com/VSCode)

    Customize VS Code with themes, formatters, language extensions and debuggers for your favorite languages, and more.

1. [Enable AI features](/docs/copilot/setup-simplified.md)

    > [!TIP]
    > If you don't yet have a Copilot subscription, you can use Copilot for free by signing up for the [Copilot Free plan](https://github.com/github-copilot/signup) and get a monthly limit of completions and chat interactions.

1. [Get started with the VS Code tutorial](/docs/getstarted/getting-started.md)

    Discover the user interface and key features of VS Code.

## Update cadence

VS Code releases a new version [each month](/updates) with new features and important bug fixes. Most platforms support auto updating and you are prompted to install the new release when it becomes available.

You can also manually check for updates by running **Help** > **Check for Updates** on Linux and Windows, or running **Code** > **Check for Updates** on macOS.

> [!NOTE]
> You can [disable auto-update](/docs/supporting/faq.md#how-do-i-opt-out-of-vs-code-autoupdates) if you prefer to update VS Code on your own schedule.

## Insiders nightly build

If you'd like to try our nightly builds to see new features early or verify bug fixes, you can install our [Insiders build](/insiders). The Insiders build installs side-by-side with the monthly Stable build and you can freely work with either on the same machine. The Insiders build is the same one the VS Code development team uses on a daily basis and we really appreciate people trying out new features and providing feedback.

## Portable mode

Visual Studio Code supports [Portable mode](https://en.wikipedia.org/wiki/Portable_application) installation. This mode enables all data created and maintained by VS Code to live near itself, so it can be moved around across environments, for example, on a USB drive. See the [VS Code Portable Mode](/docs/editor/portable.md) documentation for details.

## Next steps

Once you have installed VS Code, these topics will help you learn more about it:

* [VS Code tutorial](/docs/getstarted/getting-started.md) - A quick hands-on tour of the key features of VS Code.
* [Tips and Tricks](/docs/getstarted/tips-and-tricks.md) - A collection of productivity tips for working with VS Code.
* [AI-assisted coding](/docs/copilot/overview.md) - Learn about using GitHub Copilot in VS Code to help you write code faster.

## Common questions

### What are the system requirements for VS Code?

We have a list of [System Requirements](/docs/supporting/requirements.md).

### How big is VS Code?

VS Code is a small download (< 200 MB) and has a disk footprint of less than 500 MB, so you can quickly install VS Code and try it out.

### How do I create and run a new project?

VS Code doesn't include a traditional **File** > **New Project** dialog or pre-installed project templates. You'll need to add [additional components](/docs/setup/additional-components.md) and scaffolders depending on your development interests. With scaffolding tools like [Yeoman](https://yeoman.io/) and the multitude of modules available through the [npm](https://www.npmjs.com/) package manager, you're sure to find appropriate templates and tools to create your projects.

### How do I know which version I'm running?

On Linux and Windows, choose **Help** > **About**. On macOS, use **Code** > **About Visual Studio Code**.

### Why is VS Code saying my installation is unsupported?

VS Code has detected that some installation files have been modified, perhaps by an extension. Reinstalling VS Code will replace the affected files. See our [FAQ topic](/docs/supporting/faq.md#installation-appears-to-be-corrupt-unsupported) for more details.

### How can I do a 'clean' uninstall of VS Code?

If you want to remove all user data after [uninstalling](/docs/setup/uninstall.md) VS Code, you can delete the user data folders `Code` and `.vscode`. This returns you to the state before you installed VS Code. This can also be used to reset all settings if you don't want to uninstall VS Code.

The folder locations vary depending on your platform:

* **Windows** - Delete `%APPDATA%\Code` and `%USERPROFILE%\.vscode`.
* **macOS** - Delete `$HOME/Library/Application Support/Code` and `~/.vscode`.
* **Linux** - Delete `$HOME/.config/Code` and `~/.vscode`.
