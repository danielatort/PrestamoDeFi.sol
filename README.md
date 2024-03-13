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
        uint256 indexed prestamoId,
        address indexed prestatario, 
        uint256 monto, 
        uint256 plazo);

    event PrestamoAprobado(uint256 indexed prestamoId, address prestatario);

    event PrestamoReembolsado(uint256 indexed prestamoId, address prestatario);

    event GarantiaLiquidada(uint256 indexed prestamoId, address prestatario);

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
    
    function altaPrestamista(address nuevoPrestamista) public soloSocioPrincipal {
        require(!empleadosPrestamista[nuevoPrestamista], "El prestamista ya esta dado de alta");
        empleadosPrestamista[nuevoPrestamista] = true;
    }

    function altaCliente(address nuevoCliente) public soloEmpleadoPrestamista {
        require(!clientes[nuevoCliente].activado, "El cliente ya esta registrado");
        Cliente storage structNuevoCliente = clientes[nuevoCliente];
        structNuevoCliente.saldoGarantia = 0;
        structNuevoCliente.activado = true;
    }

    function depositarGarantia(uint256 monto_) public payable soloClienteRegistrado {
        require (msg.value> 0, "ERROR: No hay fondos a depositar");
        clientes[msg.sender].saldoGarantia += monto_;
    }

    function solicitarPrestamo(uint256 monto_, uint256 plazo_) public soloClienteRegistrado returns (uint256) {
        require(clientes[msg.sender].saldoGarantia >= monto_, "ERROR: Saldo de garantía insuficiente");
        uint256 nuevoId = clientes[msg.sender].prestamoIds.length + 1;

        Prestamo storage nuevoPrestamo = clientes[msg.sender].prestamos[nuevoId];
        nuevoPrestamo.id = nuevoId;
        nuevoPrestamo.prestatario = msg.sender;
        nuevoPrestamo.monto = monto_;
        nuevoPrestamo.plazo = plazo_;
        nuevoPrestamo.tiempoSolicitud = block.timestamp;
        nuevoPrestamo.tiempoLimite = 0; 
        nuevoPrestamo.aprobado = false;
        nuevoPrestamo.reembolsado = false;
        nuevoPrestamo.liquidado = false;

        clientes[msg.sender].prestamoIds.push(nuevoId);

        emit SolicitudPrestamo(nuevoId, msg.sender, monto_, plazo_);

        return nuevoId;
    }

    function aprobarPrestamo(address prestatario_, uint256 id_) public soloEmpleadoPrestamista {
        Cliente storage prestatario = clientes[prestatario_];
        require(id_ > 0 && prestatario.prestamoIds.length >= id_, "ERROR: Préstamo no encontrado");

        Prestamo storage prestamo = prestatario.prestamos[id_];
        require(!prestamo.aprobado, "ERROR: Préstamo ya aprobado");
        require(!prestamo.reembolsado, "ERROR: Préstamo ya reembolsado");
        require(!prestamo.liquidado, "ERROR: Préstamo ya liquidado");

        prestamo.aprobado = true;

        prestamo.tiempoLimite = block.timestamp + prestamo.plazo;

        emit PrestamoAprobado(id_, prestatario_, prestatario.saldoGarantia);
    }

    function reembolsarPrestamo(uint256 id_) public soloClienteRegistrado {
        Cliente storage prestatario = clientes[msg.sender];
        require(id_ > 0 && prestatario.prestamoIds.length >= id_, "ERROR: Préstamo no encontrado");
        
        Prestamo storage prestamo = prestatario.prestamos[id_];
        require(msg.sender == prestamo.prestatario, "ERROR: No eres el prestatario del préstamo");
        require(prestamo.aprobado, "ERROR: Préstamo no aprobado");
        require(!prestamo.reembolsado, "ERROR: Préstamo ya reembolsado");
        require(!prestamo.liquidado, "ERROR: Préstamo liquidado");
        require(prestamo.tiempoLimite >= block.timestamp, "ERROR: Tiempo límite vencido");

        _socioPrincipal.transfer(prestamo.monto);

        prestamo.reembolsado = true;
        prestatario.saldoGarantia -= prestamo.monto;

        emit PrestamoReembolsado(id_, msg.sender, prestamo.monto, prestatario.saldoGarantia);
    }
    
    function liquidarGarantia(address prestatario_, uint256 id_) public soloEmpleadoPrestamista {
        Cliente storage prestatario = clientes[prestatario_];
        require(id_ > 0 && prestatario.prestamoIds.length >= id_, "ERROR: Préstamo no encontrado");

        Prestamo storage prestamo = prestatario.prestamos[id_];
        require(prestamo.aprobado, "ERROR: Préstamo no aprobado");
        require(!prestamo.reembolsado, "ERROR: Préstamo ya reembolsado");
        require(!prestamo.liquidado, "ERROR: Préstamo ya liquidado");
        require(prestamo.tiempoLimite < block.timestamp, "ERROR: Tiempo límite no vencido");

        _socioPrincipal.transfer(prestamo.monto);

        prestamo.liquidado = true;
        
        prestatario.saldoGarantia -= prestamo.monto;

        emit GarantiaLiquidada(id_, prestatario_, prestamo.monto, prestatario.saldoGarantia);
    }

    function obtenerPrestamosPorPrestatario(address prestatario_) public view returns (uint256[] memory) {
        return clientes[prestatario_].prestamoIds;
    }

    function obtenerDetalleDePrestamo(address prestatario_, uint256 id_) public view returns (Prestamo memory) {
        return clientes[prestatario_].prestamos[id_];
    }
    
}
