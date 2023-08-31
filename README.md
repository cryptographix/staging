import { sha1 } from "https://denopkg.com/chiefbiiko/sha1/mod.ts";

export class TableDecoder {

  static decodeHeader(reg: string): any {
    let decoder = new TableDecoder(reg);
    let tab: any = {};

    let dataLen = parseInt(decoder.readAsNumber(3));
    if (dataLen + 3 < reg.length)
      throw new Error("Register length " + dataLen + " too large");

    tab.ID = decoder.readAsNumber(1);
    tab.ACQ = decoder.readAsNumber(2);
    tab.RECIDX = decoder.readAsNumber(2);

    return tab;
  }

  static decodeAID(reg: string): any {
    let decoder = new TableDecoder(reg);
    let tab: any = {};

    let dataLen = parseInt(decoder.readAsNumber(3));
    if ((dataLen != 284) && (dataLen != 314))
      throw new Error("Register length " + dataLen + " not valid");

    if (decoder.read(1) != "1")
      throw new Error("Not an AID table");

    tab.RECLEN = dataLen;

    tab.ACQ = decoder.readAsNumber(2);
    tab.RECIDX = decoder.readAsNumber(2);

    let aidLen = parseInt(decoder.readAsNumber(2));
    tab.AID = decoder.read(2 * aidLen);
    decoder.skip(32 - 2 * aidLen);

    tab.APPTYPE = decoder.readAsNumber(2);
    tab.DEFLABEL = decoder.read(16);
    tab.ICCSTD = decoder.readAsNumber(2);

    tab.APPVER1 = decoder.readAsHex(4);
    tab.APPVER2 = decoder.readAsHex(4);
    tab.APPVER3 = decoder.readAsHex(4);

    tab.TRMCNTRY = decoder.readAsNumber(3);
    tab.TRMCURR = decoder.readAsNumber(3);
    tab.TRMCRREXP = decoder.readAsNumber(1);
    tab.MERCHID = decoder.read(15);
    tab.MCC = decoder.readAsNumber(4);
    tab.TRMID = decoder.read(8);
    tab.TRMCAPAB = decoder.readAsHex(6);
    tab.ADDTRMCP = decoder.readAsHex(10);
    tab.TRMTYP = decoder.readAsNumber(2);
    tab.TACDEF = decoder.readAsHex(10);
    tab.TACDEN = decoder.readAsHex(10);
    tab.TACONL = decoder.readAsHex(10);

    tab.FLRLIMIT = decoder.readAsHex(8);
    tab.TCC = decoder.readAsHex(1);
    tab.CTLSZEROAM = decoder.readAsHex(1);
    tab.CTLSMODE = decoder.readAsHex(1);
    tab.CTLSTRNLIM = decoder.readAsHex(8);
    tab.CTLSFLRLIM = decoder.readAsHex(8);
    tab.CTLSCVMLIM = decoder.readAsHex(8);
    tab.CTLSAPPVER = decoder.readAsHex(4);
    decoder.skip(1)
    tab.TDOLDEF = decoder.readAsHex(40);
    tab.DDOLDEF = decoder.readAsHex(40);
    tab.ARCOFFLN = decoder.read(8);
    if (dataLen > 284) {
      tab.CTLSTACDEF = decoder.readAsHex(10);
      tab.CTLSTACDEN = decoder.readAsHex(10);
      tab.CTLSTACONL = decoder.readAsHex(10);
    }

    return tab;
  }

  static decodeCAPK(reg: string): any {
    let decoder = new TableDecoder(reg);
    let tab: any = {};

    let dataLen = parseInt(decoder.readAsNumber(3));
    if ((dataLen != 611))
      throw new Error("Register length " + dataLen + " not valid");

    if (decoder.read(1) != "2")
      throw new Error("Not an CAPK table");

    tab.RECLEN = dataLen;

    tab.ACQ = decoder.readAsNumber(2);
    tab.RECIDX = decoder.readAsNumber(2);

    tab.AID = decoder.read(2 * 5);
    tab.CAPKINDEX = decoder.read(2);

    const expLen = parseInt(decoder.readAsNumber(3));
    tab.EXP = decoder.read(2 * expLen);
    decoder.skip(2 * (3 - expLen));

    const modLen = parseInt(decoder.readAsNumber(3));
    tab.MOD = decoder.read(2 * modLen);
    decoder.skip(2 * (248 - modLen));

    tab.HASHTYPE = decoder.readAsNumber(1);
    tab.HASH = decoder.readAsHex( 2 * 32 );

    tab.__SHA1 = (sha1(tab.AID+tab.CAPKINDEX+tab.MOD+tab.EXP,"hex","hex") as string).toUpperCase();
    return tab;
  }


  private reg: string;
  private offset: number = 0;

  constructor(reg: string) {
    this.reg = reg;
    this.rewind();
  }

  rewind() {
    this.offset = 0;
  }

  skip(len: number) {
    this.offset += len;
  }

  read(len: number): string {
    let val = this.reg.substring(this.offset, this.offset + len);
    //    console.log( "read '" + val + "'");
    this.offset += len;

    return val;
  }

  readAsHex(len: number): string {
    return this.read(len);
  }

  readAsNumber(len: number): string {
    return this.read(len);
  }

  /*  readAsNumber( len: number ): number {
      let num = parseInt( this.read(len) );
  
      return num;
    }*/
}
