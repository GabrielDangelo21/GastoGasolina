# Gasto Gasolina — Protótipo

Este repositório contém um protótipo simples (frontend estático) que usa Google Apps Script para ler/escrever uma Google Sheet. O objetivo é controlar o direito de €75/semana e registar quanto colocaste.

Arquivos criados:
- `index.html` — frontend estático para subir no Netlify.

=== Apps Script (Code.gs)

Cole este código no editor do Apps Script (Extensions → Apps Script) ligado à tua planilha. Substitui `SPREADSHEET_ID` e `SHEET_NAME`.

function setApiToken(t){
  PropertiesService.getScriptProperties().setProperty('API_TOKEN', t);
  return 'token set';
}

function _openSheet(){
  var id = 'SPREADSHEET_ID'; // substituir pelo id da sheet
  var name = 'SHEET_NAME'; // substituir pelo nome da folha (ex: 'Sheet1')
  return SpreadsheetApp.openById(id).getSheetByName(name);
}

function getData(){
  var sheet = _openSheet();
  var rows = sheet.getDataRange().getValues();
  var headers = rows.shift();
  return rows.map(function(r,i){
    var obj = {};
    headers.forEach(function(h,j){ obj[h]=r[j]; });
    obj._rowNumber = i+2;
    return obj;
  });
}

function doGet(e){
  var token = PropertiesService.getScriptProperties().getProperty('API_TOKEN');
  if(e.parameter.token && e.parameter.token === token){
    var out = { success:true, data:getData() };
    return ContentService.createTextOutput(JSON.stringify(out)).setMimeType(ContentService.MimeType.JSON);
  } else {
    var out = { success:false, error:'invalid_token' };
    return ContentService.createTextOutput(JSON.stringify(out)).setMimeType(ContentService.MimeType.JSON);
  }
}

function doPost(e){
  try{
    var body = JSON.parse(e.postData.contents);
    var token = PropertiesService.getScriptProperties().getProperty('API_TOKEN');
    if(body.token !== token) return ContentService.createTextOutput(JSON.stringify({success:false, error:'invalid_token'})).setMimeType(ContentService.MimeType.JSON);

    var sheet = _openSheet();
    if(body.action === 'append'){
      // espera campos: Semana, GASOLINA, COLOCADO, DIFERENCA
      sheet.appendRow([body.Semana || '', body.GASOLINA || '', body.COLOCADO || '', body.DIFERENCA || '']);
      return ContentService.createTextOutput(JSON.stringify({success:true})).setMimeType(ContentService.MimeType.JSON);
    } else if(body.action === 'update'){
      var row = Number(body._rowNumber);
      sheet.getRange(row,1,1,4).setValues([[body.Semana, body.GASOLINA, body.COLOCADO, body.DIFERENCA]]);
      return ContentService.createTextOutput(JSON.stringify({success:true})).setMimeType(ContentService.MimeType.JSON);
    } else {
      return ContentService.createTextOutput(JSON.stringify({success:false, error:'unknown_action'})).setMimeType(ContentService.MimeType.JSON);
    }
  } catch(err){
    return ContentService.createTextOutput(JSON.stringify({success:false, error:err.message})).setMimeType(ContentService.MimeType.JSON);
  }
}

=== Passos para configurar

1. Abre a tua Google Sheet. Vai a Extensões → Apps Script.
2. Cria um novo projecto e cola o código acima em `Code.gs`.
3. Substitui `SPREADSHEET_ID` pelo id da tua sheet (parte da URL entre `/d/` e `/edit`).
4. Substitui `SHEET_NAME` pelo nome da folha onde tens os dados (ex: `Sheet1` ou `Página1`).
5. No editor do Apps Script, executa a função `setApiToken('uma_senha_forte_aqui')` uma vez — autoriza quando pedido.
6. Deploy → New deployment → Tipo: Web app → Execute as: Me → Who has access: Anyone (ou Anyone, even anonymous) → Deploy. Copia a URL de execução (a usar no `index.html`).

Nota: para chamadas diretas do browser, a opção "Anyone" costuma funcionar. Se preferires evitar qualquer exposição pública do endpoint, usas Netlify Functions como proxy e manténs o Apps Script mais restrito.

=== Como usar o frontend

1. Abre `index.html` com um editor ou sube o ficheiro para o teu repositório Netlify.
2. Põe a Apps Script URL e o token nos campos na página.
3. Carrega os dados com o botão "Carregar dados".
4. Adiciona uma nova semana preenchendo `Semana` e `Colocado` e clicando em `Adicionar`.

=== Limitações e segurança

- O script usa um token simples guardado em `PropertiesService` para autorizar chamadas. Não é para dados sensíveis.
- Se precisares de mais segurança, implementa um proxy no Netlify e guarda as credenciais como variables de ambiente.
- Sheets não é um banco de dados transaccional — para muitos utilizadores a escrita concorrente pode causar condições de corrida.

Se quiseres, eu posso criar um repositório Git com estes ficheiros e instruções para push e deploy no Netlify.
