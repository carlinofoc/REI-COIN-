// SPDX-License-Identifier: MIT
// ------------------------------------------------
//          REI COIN - VERSÃO FINAL E CORRIGIDA
// LIMITE DE VENDA/TRANSFERÊNCIA: R$ 3.000,00 por dia
// Rede: Polygon | Padrão: ERC-20 Oficial
// Carteira do Dono: 0xF968abfFCEF604AE5a6931E0e826667C6812245e
// Regras: Imutáveis, sem criação de moedas extras
// ------------------------------------------------

pragma solidity ^0.8.20;

contract REICOIN {
    string public constant NOME = "REI COIN";
    string public constant SIMBOLO = "REI";
    uint8 public constant CASAS_DECIMAIS = 18;

    // ✅ Quantidade total fixa: 3 MIL TRILHÕES DE REI
    uint256 public constant VALOR_TOTAL = 3_000_000_000_000_000 * 10 ** CASAS_DECIMAIS;

    // ✅ Endereço do Dono
    address public constant DONO = address(0xF968abfFCEF604AE5a6931E0e826667C6812245e);

    // ✅ Limite de transação diária
    uint256 public constant LIMITE_REAIS = 3_000 * 10 ** CASAS_DECIMAIS;
    uint256 public constant TAXA_CONVERSAO = 1 * 10 ** CASAS_DECIMAIS;

    // ✅ Taxa de rede (0,1% por operação)
    uint256 public constant TAXA_TRANSACAO = 10; // 10 = 0,1% (divide por 10.000)
    address public constant CARTEIRA_TAXAS = address(0xF968abfFCEF604AE5a6931E0e826667C6812245e);

    mapping(address => uint256) private _saldos;
    mapping(address => mapping(address => uint256)) private _permissoes;

    // ✅ Controle de limite por dia
    mapping(address => uint256) private _usadoHoje;
    mapping(address => uint256) private _ultimoDia;

    // ✅ BLOQUEIO DE ENDEREÇOS
    mapping(address => bool) public enderecoBloqueado;

    // ✅ NÍVEL DE USUÁRIO: 0 = Comum | 1 = Parceiro | 2 = Administrador
    mapping(address => uint8) public nivelUsuario;

    // ✅ HISTÓRICO COMPLETO DE MOVIMENTAÇÕES
    struct Movimentacao {
        string tipo;
        uint256 valor;
        uint256 valorTaxa;
        uint256 dataHora;
        address de;
        address para;
        string observacao;
    }
    mapping(address => Movimentacao[]) public historico;

    event Transfer(address indexed de, address indexed para, uint256 valor);
    event Approval(address indexed dono, address indexed gastador, uint256 valor);
    event EnderecoBloqueado(address indexed conta, string motivo);
    event EnderecoDesbloqueado(address indexed conta);
    event NivelAlterado(address indexed conta, uint8 novoNivel);

    constructor() {
        _saldos[DONO] = VALOR_TOTAL;
        nivelUsuario[DONO] = 2; // Dono tem nível máximo
        emit Transfer(address(0), DONO, VALOR_TOTAL);
    }

    // --- FUNÇÕES BÁSICAS ---
    function totalSupply() public pure returns (uint256) {
        return VALOR_TOTAL;
    }

    function balanceOf(address conta) public view returns (uint256) {
        return _saldos[conta];
    }

    function saldoEmReais(address conta) public view returns (uint256) {
        return _saldos[conta] / TAXA_CONVERSAO;
    }

    // ✅ VERIFICA QUANTO PODE USAR NO DIA
    function verLimiteDoDia(address usuario) public view returns (uint256 usado, uint256 restante) {
        if (nivelUsuario[usuario] >= 2) {
            return (0, LIMITE_REAIS * 100); // Administrador sem limite
        }
        uint256 diaAtual = block.timestamp / 86400;
        if (_ultimoDia[usuario] != diaAtual) {
            return (0, LIMITE_REAIS);
        }
        return (_usadoHoje[usuario], LIMITE_REAIS - _usadoHoje[usuario]);
    }

    // ✅ BLOQUEAR E DESBLOQUEAR CONTAS
    function bloquearEndereco(address conta, string calldata motivo) public {
        require(msg.sender == DONO, "Apenas o dono pode executar esta acao");
        require(conta != DONO, "Nao e permitido bloquear a conta do dono");
        enderecoBloqueado[conta] = true;
        emit EnderecoBloqueado(conta, motivo);
    }

    function desbloquearEndereco(address conta) public {
        require(msg.sender == DONO, "Apenas o dono pode executar esta acao");
        enderecoBloqueado[conta] = false;
        emit EnderecoDesbloqueado(conta);
    }

    // ✅ ALTERAR NÍVEL DE ACESSO
    function alterarNivel(address conta, uint8 novoNivel) public {
        require(msg.sender == DONO, "Apenas o dono pode executar esta acao");
        require(novoNivel <= 2, "Nivel maximo permitido e 2");
        nivelUsuario[conta] = novoNivel;
        emit NivelAlterado(conta, novoNivel);
    }

    // ✅ TRANSFERÊNCIA PRINCIPAL — AGORA COM string memory PARA COMPATIBILIDADE
    function transfer(address para, uint256 valor, string memory observacao) public returns (bool) {
        require(!enderecoBloqueado[msg.sender], "Conta de origem esta bloqueada");
        require(!enderecoBloqueado[para], "Conta de destino esta bloqueada");
        require(para != address(0), "Endereco de destino invalido");
        require(para != msg.sender, "Nao e permitido transferir para si mesmo");
        require(valor > 0, "Valor deve ser maior que zero");
        require(valor <= _saldos[msg.sender], "Saldo insuficiente para operacao");

        // Controle de limite diário
        if (nivelUsuario[msg.sender] < 2) {
            uint256 diaAtual = block.timestamp / 86400;
            if (_ultimoDia[msg.sender] != diaAtual) {
                _usadoHoje[msg.sender] = 0;
                _ultimoDia[msg.sender] = diaAtual;
            }
            require(_usadoHoje[msg.sender] + valor <= LIMITE_REAIS, "Valor ultrapassa o limite diario de R$ 3.000,00");
            _usadoHoje[msg.sender] += valor;
        }

        // Cálculo da taxa
        uint256 taxa = (valor * TAXA_TRANSACAO) / 10000;
        uint256 valorFinal = valor - taxa;

        // Executa a movimentação
        _saldos[msg.sender] -= valor;
        _saldos[para] += valorFinal;
        if (taxa > 0) {
            _saldos[CARTEIRA_TAXAS] += taxa;
        }

        // Registra no histórico
        historico[msg.sender].push(Movimentacao("Saida", valor, taxa, block.timestamp, msg.sender, para, observacao));
        historico[para].push(Movimentacao("Entrada", valorFinal, taxa, block.timestamp, msg.sender, para, observacao));

        emit Transfer(msg.sender, para, valorFinal);
        return true;
    }

    // ✅ Versão simplificada — AGORA FUNCIONA SEM ERRO
    function transfer(address para, uint256 valor) public returns (bool) {
        return transfer(para, valor, "");
    }

    function approve(address gastador, uint256 valor) public returns (bool) {
        require(!enderecoBloqueado[msg.sender], "Conta bloqueada");
        _permissoes[msg.sender][gastador] = valor;
        emit Approval(msg.sender, gastador, valor);
        return true;
    }

    function allowance(address dono_, address gastador) public view returns (uint256) {
        return _permissoes[dono_][gastador];
    }

    // ✅ transferFrom também corrigido
    function transferFrom(address de, address para, uint256 valor, string memory observacao) public returns (bool) {
        require(!enderecoBloqueado[de] && !enderecoBloqueado[para] && !enderecoBloqueado[msg.sender], "Uma das contas esta bloqueada");
        require(para != address(0) && para != de, "Endereco invalido ou repetido");
        require(valor > 0 && valor <= _saldos[de] && valor <= _permissoes[de][msg.sender], "Valor ou permissao insuficiente");

        if (nivelUsuario[de] < 2) {
            uint256 diaAtual = block.timestamp / 86400;
            if (_ultimoDia[de] != diaAtual) {
                _usadoHoje[de] = 0;
                _ultimoDia[de] = diaAtual;
            }
            require(_usadoHoje[de] + valor <= LIMITE_REAIS, "Valor ultrapassa o limite diario");
            _usadoHoje[de] += valor;
        }

        uint256 taxa = (valor * TAXA_TRANSACAO) / 10000;
        uint256 valorFinal = valor - taxa;

        _saldos[de] -= valor;
        _saldos[para] += valorFinal;
        if (taxa > 0) {
            _saldos[CARTEIRA_TAXAS] += taxa;
        }
        _permissoes[de][msg.sender] -= valor;

        historico[de].push(Movimentacao("Saida", valor, taxa, block.timestamp, de, para, observacao));
        historico[para].push(Movimentacao("Entrada", valorFinal, taxa, block.timestamp, de, para, observacao));

        emit Transfer(de, para, valorFinal);
        return true;
    }

    function transferFrom(address de, address para, uint256 valor) public returns (bool) {
        return transferFrom(de, para, valor, "");
    }

    // ✅ CONSULTAS DO HISTÓRICO
    function quantidadeMovimentacoes(address conta) public view returns (uint256) {
        return historico[conta].length;
    }

    function verMovimentacaoCompleta(address conta, uint256 indice) public view returns (
        string memory tipo,
        uint256 valorTotalReais,
        uint256 taxaReais,
        string memory dataHora,
        address de,
        address para,
        string memory observacao
    ) {
        Movimentacao memory mov = historico[conta][indice];
        uint256 dias = mov.dataHora / 86400;
        uint256 horas = (mov.dataHora % 86400) / 3600;
        return (
            mov.tipo,
            mov.valor / TAXA_CONVERSAO,
            mov.valorTaxa / TAXA_CONVERSAO,
            string(abi.encodePacked("Dia: ", uint2str(dias), " | Hora: ", uint2str(horas), "h")),
            mov.de,
            mov.para,
            mov.observacao
        );
    }

    // ✅ Função auxiliar corrigida
    function uint2str(uint256 _i) internal pure returns (string memory) {
        if (_i == 0) {
            return "0";
        }
        uint256 j = _i;
        uint256 comprimento;
        while (j != 0) {
            comprimento++;
            j /= 10;
        }
        bytes memory b = new bytes(comprimento);
        while (_i != 0) {
            comprimento--;
            b[comprimento] = bytes1(uint8(48 + _i % 10));
            _i /= 10;
        }
        return string(b);
    }
}
