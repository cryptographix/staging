import { TableDecoder } from './table-decoder.ts';

// let fd = fs.openSync( ".\\bsdte3l_prod.txt", 'r' );

let file = (Deno.args.length >= 2 ) ? Deno.args[2] : ".\\bsdte2l_prod.txt";

let data = Deno.readFileSync(file);
//console.log( data );

let lines = new TextDecoder().decode(data).split('\n');

console.log( lines.length );

interface TableReg {
  source: string;
  dataReg: string;
  scheme: number;
  profile: number;
  index: number;
  appParams: { [index: string]: string }
}

let info: {
  regs: TableReg[];
  profiles: number[];
  schemes: number[];
} = {
  regs: [],
  profiles: [],
  schemes: [],
}

for( let line of lines ) {
  if ( line.startsWith( "BNJ") && !line.startsWith( "BNJF00001" )) {
    let sansHeader = line.slice( "BNJF00002                                 00".length );

    if ( sansHeader[ 12 ] == "1" ) { // AID
      let dataReg = sansHeader.slice( 9 );
      let reg: TableReg;

      let scheme = Number.parseInt( sansHeader.substr( 3, 3 ) );
      let profile = Number.parseInt( sansHeader.substr( 6, 3 ) );

      info.schemes[ scheme ] = ( info.schemes[ scheme ] ?? 0 ) + 1;
      info.profiles[ profile ] = ( info.profiles[ profile ] ?? 0 ) + 1;

      let appParams = TableDecoder.decodeAID( dataReg );

      reg = {
        source: sansHeader,
        dataReg,
        scheme,
        profile,
        index: Number.parseInt( appParams.RECIDX ),
        appParams
       };

      info.regs.push( reg );
    }
  }
}


/**
 * Check that Indexes are unique within each profile
 */
let indexByProfile = new Map<number,number[]>();

info.regs.forEach( reg => {
  let profile = reg.profile;

  if ( !indexByProfile.has( profile ) )
    indexByProfile.set( profile, [] );

  indexByProfile.get( profile )?.push( reg.index );
});

indexByProfile.forEach( ( indexes, profile ) => {
  if ( arrayHasDuplicates( indexes ) ) {
    console.log( "** Profile " + profile + " has duplicate indexes ");
    console.log( indexes.join( ',') )
  }
})

/**
 * Validate each AID
 */
let byAID: Map<string, TableReg[]> = new Map<string, TableReg[]>();

info.regs.forEach( reg => {
  let aid = reg.appParams.AID;

  if ( !byAID.has( aid ) )
    byAID.set( aid, [] );

  byAID.get( aid )?.push( reg );
})

let str = "AID ".padEnd( 20, " ")
        + "LABEL".padEnd( 16, " ");

for( let profile in info.profiles ) {
  str += "  " + (profile + "").padEnd( 2 );
}
str += " PMT CTL"

//console.log( str );

byAID.forEach( (regs, aid ) => {
  let str = aid.padEnd( 20, " ") + regs[0].appParams.DEFLABEL.padEnd(16);

  for( let profile in info.profiles ) {
    let has = regs.some( reg => {
//      console.log( reg.scheme )
      return reg.profile == Number.parseInt(profile);
    });

    str += (has) ? "  X " : "  . ";
  }

  let schemes = regs
    .map( (reg) => reg.scheme)
    .filter( (item, pos, arr) => {
      return arr.indexOf(item)== pos;
    })
    .map( (scheme) => scheme.toString().padStart(3, "0") )
    .join( ",");

  str += " " + schemes;

  let ctmodes = regs
    .map( (reg) => reg.appParams.CTLSMODE)
    .filter( (item, pos, arr) => {
      return arr.indexOf(item)== pos && (item != " ");
    })
    .map( (ctmode) => ctmode.toString() )
    .join( ",");

  str += " " + ctmodes;

//  console.log( str );

  let duplProfile = regs.some( (reg, pos, regs) => {
    return regs.some( (reg2, pos2) => {
      return pos != pos2
          && reg.profile == reg2.profile;
    } )
  });
  if ( duplProfile ) {
    console.log( "^^^^DUPLICATE PROFILE^^^^")
  }

  let inconsScheme = regs.some( (reg, pos, regs) => {
    return regs.some( (reg2, pos2) => {
      return pos != pos2
          && reg.scheme != reg2.scheme;
    } )
  });
  if ( inconsScheme ) {
    console.log( "^^^^SCHEME INCONSISTENT^^^^")
  }

  let firstDiff = true;

/*  let diffRegs = regs.filter( ( reg, index ) => {
    let baseReg = regs[1].dataReg.slice(8, Number.parseInt( regs[1].appParams.RECLEN ) );
    let thisReg = reg.dataReg.slice(8, Number.parseInt( reg.appParams.RECLEN ) );

    if ( index != 1 ) {
      if ( firstDiff ) {
        firstDiff = false;
        console.log( regs[1].profile.toString().padStart(3,'0') + ":" + baseReg );
      }

      console.log( "    " + Array.from(thisReg).map( (ch,index) => {
        return ( baseReg[ index ] == ch ) ? ' ' : '*';
      }).join( '' ))
      
      console.log( reg.profile.toString().padStart(3,'0') + ":" + thisReg );
    }

    return ( index > 0 ) && baseReg != thisReg;
  } )

  if ( diffRegs.length > 0 ) {
    console.log( "^^^^REGISTERS DIFFER^^^^" + diffRegs.map( (reg) => reg.profile ).join(','))
  }*/

})

// Check Indexes
//info.regs.


function arrayHasDuplicates<U>( arr: U[] ) {
  return arr.length !== new Set(arr).size;
}
