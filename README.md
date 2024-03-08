// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PrestamoDeFi {

    address private _socioPrincipal;

    struct Prestamo {
        uint256 id;
        address prestatario;
        uint256 monto;
        uint256 plazo;
        uint256 tiempoSolicitud;
        uint256 tiempoLimite;
        bool aprobado;
        bool reembolsado;
        bool liquidado;
    }

    struct Cliente {
        bool activado;
        uint256 saldoGarantia;
        mapping(uint256 => Prestamo) prestamos;
        uint256[] prestamoIds;
    }

    mapping(address => Cliente) public clientes;

    mapping(address => bool) public empleadosPrestamista;

    event SolicitudPrestamo(
        uint256 prestamoId,
        address prestatario, 
        uint256 monto, 
        uint256 plazo);

    event PrestamoAprobado(uint256 prestamoId, address prestatario);

    event PrestamoReembolsado(uint256 prestamoId, address prestatario);

    event GarantiaLiquidada(uint256 prestamoId, address prestatario);

    modifier soloSocioPrincipal() {
        require(msg.sender == _socioPrincipal, "ERROR: No eres la persona autorizada");
        _;
    }

    modifier soloEmpleadoPrestamista() {
        require(empleadosPrestamista[msg.sender], "ERROR: No eres la persona autorizada");
        _;
    }

    modifier soloClienteRegistrado() {
        require(clientes[msg.sender].activado, "ERROR: Cliente no registrado o inactivo");
        _;
    }

    constructor(){
        _socioPrincipal = msg.sender;
    }
    
    function altaPrestamista(address nuevoPrestamista_) public soloSocioPrincipal {
        empleadosPrestamista[nuevoPrestamista_] = true;
    }
    
    function altaCliente(address nuevoCliente_) public soloEmpleadoPrestamista{
        clientes[nuevoCliente_].activado = true;
    }

    function depositarGarantia(uint256 monto_) public payable soloClienteRegistrado {
        require (msg.value> 0, "ERROR: No hay fondos a depositar");
        clientes[msg.sender].saldoGarantia += monto_;
    }

   

}
